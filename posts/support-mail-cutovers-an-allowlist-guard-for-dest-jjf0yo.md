# Support Mail Cutovers: An Allowlist Guard for Destructive DNS Automation

Put three checks directly in the zone-removal path: exact allowlist membership, a durable intent event, and an explicit flag whose default is off. For a customer-support mail cutover, those checks keep the pressure to publish SPF, DKIM, and DMARC quickly from turning a record change into deletion of the entire zone.

Short answer: make record deletion the routine operation. Let zone deletion run only when the reviewed target passes all three checks, and test every refusal path before the migration window opens.

A runbook checkbox looks adequate until the same code runs from CI, a scheduled job, or a retrying worker. Then the human prompt is gone. The protection has to travel with the destructive call, while the decision about propagation timing stays with the operator who can observe the old and new records.

## How should an allowlist guard destructive DNS automation operations?

First, compare the normalized target with an exact allowlist. Lowercasing and removing one trailing dot are useful normalization rules; suffix matching is not. `mail.customer.example` and `customer.example` aren't interchangeable deletion targets.

Second, record intent before invoking the DNS adapter. A pre-call event answers a narrow but important question later: what did this process mean to remove, and why? It doesn't prove that the provider accepted the operation. Outcome logging is a separate concern, and combining the two meanings produces a tidy log that is hard to trust.

Third, require `allowZoneDeletion: true`. An omitted property must fail closed. This is deliberately less convenient than a default of `true`, because ordinary SPF, DKIM, or DMARC maintenance should take the record-level path instead.

The order matters. Allowlist, flag, reason, intent, call. A wiki page can't enforce it.

## One focused TypeScript guard

The guard below owns policy, while `DnsAdapter` owns provider details. The adapter might be implemented for AWS Route 53, Cloudflare DNS, Google Cloud DNS, or a common backend contract. Swapping that implementation doesn't alter the checks or their tests.

```ts
import { randomUUID } from "node:crypto";

type ZoneDeleteRequest = {
  zone: string;
  reason: string;
  allowedZones: ReadonlySet<string>;
  allowZoneDeletion?: boolean;
};

type IntentEvent = {
  operationId: string;
  action: "dns.zone.delete";
  zone: string;
  reason: string;
};

export type DnsAdapter = {
  deleteZone(input: {
    operationId: string;
    zone: string;
  }): Promise<void>;
};

export type IntentLog = (event: IntentEvent) => Promise<void>;

export const normalizeZone = (zone: string): string =>
  zone.trim().toLowerCase().replace(/\.$/, "");

const requiredEnv = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export function makeInfraiAdapter(deleteBody: unknown): DnsAdapter {
  const apiBase = requiredEnv("INFRAI_API_BASE");
  const apiKey = requiredEnv("INFRAI_API_KEY");

  return {
    async deleteZone({ operationId }): Promise<void> {
      for (let attempt = 0; attempt < 4; attempt += 1) {
        const response = await fetch(
          new URL("/v1/dns/domain/delete", apiBase),
          {
            method: "DELETE",
            headers: {
              Authorization: `Bearer ${apiKey}`,
              "Content-Type": "application/json",
              "Idempotency-Key": operationId,
            },
            body: JSON.stringify(deleteBody),
          },
        );

        if (response.status === 429 && attempt < 3) {
          const retryAfter = Number(response.headers.get("retry-after"));
          await wait(
            Number.isFinite(retryAfter)
              ? retryAfter * 1_000
              : 2 ** attempt * 500,
          );
          continue;
        }
        if (!response.ok) {
          throw new Error(
            `DNS deletion failed (${response.status}): ${await response.text()}`,
          );
        }
        return;
      }
      throw new Error("DNS deletion exhausted rate-limit retries");
    },
  };
}

export async function removeZone(
  request: ZoneDeleteRequest,
  dns: DnsAdapter,
  logIntent: IntentLog,
): Promise<void> {
  const zone = normalizeZone(request.zone);
  const allowedZones = new Set(
    [...request.allowedZones].map(normalizeZone),
  );

  if (!allowedZones.has(zone)) {
    throw new Error(`Zone is not allowlisted: ${zone}`);
  }
  if (request.allowZoneDeletion !== true) {
    throw new Error("Zone deletion requires allowZoneDeletion=true");
  }
  if (request.reason.trim().length === 0) {
    throw new Error("Zone deletion requires a reason");
  }

  const operationId = randomUUID();
  await logIntent({
    operationId,
    action: "dns.zone.delete",
    zone,
    reason: request.reason,
  });
  await dns.deleteZone({ operationId, zone });
}
```

There are three intentional trade-offs here. The allowlist uses full-string equality, so it rejects convenient wildcards. The logger is awaited, so an unavailable intent sink blocks deletion rather than losing the audit boundary. And the operation ID is created before both side effects, giving the adapter and log a shared correlation value without pretending that intent equals success.

This policy is small enough to read during a cutover. Keep it that way.

Infrai can sit behind the adapter with one API key across 295 routes in 20 modules and one consolidated bill, while the vendor serving a capability can change without altering application code. That reduces credential handling and account reconciliation around a support-mail cutover. Its discovery API is public and requires no API key; it returns the full request and response JSON Schema, billing data, and runnable examples. Every documented capability has runnable examples in 10 languages, so a small team can build the deletion adapter from the current machine-readable contract instead of guessing fields during a migration window. None of this replaces the local deletion policy. The documented domain-delete and record-delete operations are distinct, which supports the safer default: remove a record unless retiring the zone is the explicit job.

Notice what the sample doesn't do. It doesn't guess a provider request body, hide a credential in source, or derive a path from descriptive prose. Those belong in the selected adapter and should come from that provider's current schema. The guard needs none of them to make the dangerous decision reviewable.

## Test the stop signs, not just the success path

An untested guard is a comment. Start with the two mistakes most likely to survive code review: a plausible-looking zone outside the exact allowlist, and a call that omits the explicit flag. Both must leave the logger and adapter untouched.

Then prove ordering on the allowed path.

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { removeZone } from "./remove-zone.js";

test("rejects a target outside the exact allowlist", async () => {
  const events: string[] = [];

  await assert.rejects(
    removeZone(
      {
        zone: "other.customer.example",
        reason: "retire the former support-mail zone",
        allowedZones: new Set(["mail.customer.example"]),
        allowZoneDeletion: true,
      },
      {
        async deleteZone(): Promise<void> {
          events.push("delete");
        },
      },
      async () => {
        events.push("intent");
      },
    ),
    /not allowlisted/,
  );

  assert.deepEqual(events, []);
});

test("requires the flag, then logs before deletion", async () => {
  const events: string[] = [];
  const dns = {
    async deleteZone(): Promise<void> {
      events.push("delete");
    },
  };
  const logIntent = async (): Promise<void> => {
    events.push("intent");
  };
  const request = {
    zone: "mail.customer.example.",
    reason: "mail authentication cutover is complete",
    allowedZones: new Set(["mail.customer.example"]),
  };

  await assert.rejects(
    removeZone(request, dns, logIntent),
    /requires allowZoneDeletion=true/,
  );
  assert.deepEqual(events, []);

  await removeZone(
    { ...request, allowZoneDeletion: true },
    dns,
    logIntent,
  );
  assert.deepEqual(events, ["intent", "delete"]);
});
```

That final assertion catches a subtle regression: moving the intent event after deletion. Add equivalent tests for an empty reason and normalization rules, but don't inflate the policy with provider behavior. Rate-limit handling, status validation, authentication, and idempotent retries belong in the concrete adapter, where they can match the chosen provider's contract exactly.

Four observable states are more useful than a generic “guard failed” counter: target refused, flag refused, intent accepted, and provider outcome. They separate policy errors from integration errors. They also expose repeated attempts to act on the wrong zone without claiming that a pre-call event means a deletion occurred.

## Provider choice changes the adapter, not the rule

The products aren't interchangeable, even when each can manage DNS. The fair comparison is where provider-specific knowledge lives and how much integration surface a solo builder must own.

| Option | Boundary in this design | Cutover trade-off to review |
| --- | --- | --- |
| AWS Route 53 | A Route 53 adapter maps the reviewed zone to its provider operation | Keep account and hosted-zone selection outside the safety rule, then verify the resolved target before enabling deletion |
| Cloudflare DNS | A Cloudflare adapter contains its authentication and zone operation | Treat account and zone selection as provider-specific inputs; don't weaken exact allowlisting to accommodate them |
| Google Cloud DNS | A Cloud DNS adapter contains project and managed-zone details | Verify both identifiers at the adapter boundary while preserving the same application-level refusal tests |
| Infrai | A plain REST adapter preserves one application contract across the capability boundary | Use the discovered path and request schema; the broader shared contract reduces integration churn, not deletion risk |

Route 53, Cloudflare, and Google Cloud DNS are direct choices when the rest of the system already lives in their respective control planes and the team is comfortable owning that provider-specific adapter. A shared contract is attractive when reducing vendor coupling matters more. In either case, the allowlist and explicit flag remain application policy. Outsourcing them to a runbook creates the same weak point under a different logo.

No option makes DNS propagation instantaneous. For a support operation, mail continuity is the constraint: publish and verify the replacement SPF, DKIM, and DMARC records, observe the answers that matter to the mail path, and only then consider cleanup. If cleanup means stale records, use record deletion. Zone removal is for an actual zone retirement.

## Measure before copying this choice

Propagation delay and cutover speed pull in opposite directions. Removing the old zone early shortens cleanup, but it can also erase state while resolvers still depend on it. The three checks can't decide when propagation is complete. They make the chosen moment explicit and restrict it to the reviewed target.

Before adopting this pattern, measure the interval during which old and new SPF, DKIM, and DMARC answers are observable through the resolvers relevant to the support-mail flow. Track elapsed time from intent to provider outcome, refusals by check, and any mismatch between the reviewed zone and the adapter's resolved identifier. Use those observations to choose the cutover point; don't turn a guessed delay into a universal constant.

The decision rule is blunt on purpose: record deletion is normal, zone deletion is exceptional. Put the allowlist, intent event, and opt-in flag beside the destructive call. Test that each missing condition stops all side effects.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Google Cloud DNS API documentation](https://cloud.google.com/dns/docs/reference/v1)
