# Zone Portability in 2026: One Internal DNS Plus Mail Endpoint Explained

Treat the internal endpoint as a request to reconcile a sending-domain intent, not as a remote-control wrapper around a registrar. For a game studio moving regional zones away from a registrar-specific API, the deciding constraint is ownership: customer-owned zones need instructions and verification, while platform-owned zones can be changed automatically. One request can start both paths, but it must not pretend both finish synchronously.

TL;DR: accept a domain plus its ownership mode, persist the desired mail state, and return a stable operation resource. A worker then either applies DNS changes through a narrow provider adapter or reports the exact records the customer must publish. In both cases, verification reads public DNS before mail is enabled. This keeps the email bundle portable without making a slow or partially delegated DNS operation look atomic.

## How should one internal endpoint set up a sending domain?

DNS publication and mail authorization cross administrative boundaries. An HTTP handler can record intent, but it cannot guarantee that a customer will edit a zone, that authoritative data will be visible to a verifier at that instant, or that every provider represents changes the same way. Returning `ready` because an API call succeeded confuses acceptance with observed end state.

That distinction matters in gaming. Imagine `guild-mail.example` serving account and tournament messages across three game regions. The studio controls the parent zone used by its North American service, while publishing partners control delegated zones for Europe and Asia. A registrar-shaped implementation can write the first zone, but it cannot legitimately write either partner's zone, even though all three need the same approved mail policy. If the handler treats missing provider credentials as a transient error, it will retry work that can never succeed. If it marks the operation ready after configuring only the studio zone, mail activation races ahead of DNS. An ownership field resolves the ambiguity before any provider call: one branch may write, the other must return instructions, and both wait for the same public observation before becoming ready.

The simple design is one long request: create records, ask the mail system to verify them, and wait. It fails at the ownership boundary. It also encourages retries of the whole sequence after a timeout, which can repeat side effects and hide which step actually needs attention.

Use two phases instead. Phase 1 commits desired state and returns an operation. Phase 2 reconciles and verifies that state asynchronously.

No fake atomicity.

## Model ownership before provider details

The endpoint needs an explicit ownership choice. Do not infer it from which credentials happen to exist today; credentials rotate, partners change, and a zone can move without changing the domain's mail intent.

| Zone mode | System action | Caller receives | Ready condition |
| --- | --- | --- | --- |
| `platform` | Apply the desired record set through an adapter | Operation ID and current status | Public DNS matches the saved intent |
| `customer` | Generate the desired record set without writing the zone | Operation ID, names, types, and values | Public DNS matches the saved intent |

Both modes should converge on the same verifier. That prevents a subtle split where automated zones are trusted after a provider response while customer zones are checked from the public side. The public observation is the useful boundary because it is also what an external mail receiver can inspect.

The trade-off is extra state, a worker, and delayed completion. This pattern is a poor fit for a single zone that one team controls, changes rarely, and can configure through an existing deployment pipeline; a reviewed configuration change is easier there. I would choose the coordinator only when ownership varies, repeated tenant onboarding justifies automation, or zone portability is an actual requirement. It also cannot eliminate customer delay. The operation model makes that delay visible; it does not make someone else publish records faster.

DMARC adds another ownership decision. RFC 7489 defines a domain-owner policy published in DNS and describes aggregate and failure reporting destinations. The platform should therefore accept a deliberate policy input rather than silently choosing an enforcement policy for every tenant. Start with the policy the domain owner approved, preserve its exact desired value, and expose verification separately from publication.

## A focused TypeScript contract

This example keeps the transport small and the provider-specific machinery behind `zones.apply`. The illustrative record values are placeholders produced by a mail configuration service; they are not hard-coded claims about a commercial API.

```ts
type ZoneMode = "customer" | "platform";
type SetupStatus = "pending_dns" | "verifying" | "ready" | "failed";

type DnsRecord = {
  name: string;
  type: "TXT" | "CNAME";
  value: string;
};

type SetupRequest = {
  domain: string;
  zoneMode: ZoneMode;
  idempotencyKey: string;
  dmarcPolicy: "none" | "quarantine" | "reject";
};

type SetupOperation = {
  id: string;
  domain: string;
  status: SetupStatus;
  requiredRecords: DnsRecord[];
};

interface SetupStore {
  findByKey(key: string): Promise<SetupOperation | undefined>;
  create(input: SetupRequest, records: DnsRecord[]): Promise<SetupOperation>;
}

interface MailIntent {
  recordsFor(domain: string, policy: SetupRequest["dmarcPolicy"]): Promise<DnsRecord[]>;
}

interface ZoneWriter {
  apply(domain: string, records: DnsRecord[]): Promise<void>;
}

async function requestSendingDomain(
  input: SetupRequest,
  store: SetupStore,
  mail: MailIntent,
  zones: ZoneWriter,
  enqueueVerification: (operationId: string) => Promise<void>,
): Promise<SetupOperation> {
  const existing = await store.findByKey(input.idempotencyKey);
  if (existing) return existing;

  const records = await mail.recordsFor(input.domain, input.dmarcPolicy);
  const operation = await store.create(input, records);

  if (input.zoneMode === "platform") {
    await zones.apply(input.domain, records);
  }

  await enqueueVerification(operation.id);
  return operation;
}
```

The handler does only four durable things: deduplicate the request, obtain the desired mail records, save them, and schedule observation. For a customer-owned zone, `requiredRecords` is the handoff contract. For a platform-owned zone, the same array is input to the adapter and remains visible for diagnosis.

Do not put registrar credentials in this request body. Bind a platform-owned zone to a credential reference under separate access control, then let the adapter resolve it. Otherwise an endpoint intended to configure mail becomes a credential-ingestion surface with a much larger security and logging burden.

## Failure is a state, not an HTTP surprise

The initial response should mean “accepted and recorded.”

Later states need enough detail to act on without exposing secrets. A verifier can report a mismatched record name and type, the expected value, the observed values, and the observation time. It should distinguish a missing record from a conflicting record because the operator response differs.

Retries belong to individual steps. Reapplying an already-matching record should be harmless; rereading DNS should not recreate mail configuration; repeating the initial request with the same idempotency key should return the original operation. Put a bounded error category on the operation, such as `dns_write_denied`, `dns_not_observed`, or `mail_verification_rejected`, while keeping raw provider messages in restricted logs.

Keep the transition history. A single `failed` flag loses the difference between an immediate authorization problem and a record that remained absent across repeated observations.

This is where many otherwise tidy abstractions leak. Provider adapters may disagree on record identifiers, replacement semantics, and how they represent a fully qualified name. Keep that translation inside the adapter. The saved intent should use one canonical domain-name form, and verification should compare normalized DNS data rather than provider response objects.

## What should you measure before copying this design?

Measure elapsed time from acceptance to public observation, separated by ownership mode and zone adapter. Also track verification attempts per operation, idempotent replays, write-denied failures, mismatches by record type, and operations still pending after the team's chosen service window. Those numbers reveal whether the bottleneck is API latency, customer action, credentials, or an incorrect desired record set.

Cost follows the same boundary. Count external writes and DNS reads per completed setup. Cache only within a clearly defined observation interval; an aggressive cache can make the status endpoint cheap while delaying recognition of a correct customer change. Avoid polling from every browser session. One scheduled verifier should update durable state, and clients should read that state.

Before adopting the pattern, test four cases: a repeated idempotency key, a partial platform write, a customer publishing one wrong value, and a zone moved to a different adapter mid-operation. The last case is the portability test. If the operation can resume from saved intent without rebuilding mail state, the internal contract is doing its job.

The endpoint count is not the architectural win. The useful result is one ownership-aware workflow whose desired state survives provider changes, whose readiness is based on public evidence, and whose retries do not multiply side effects. That is the shape to keep when a game studio moves zones, partners, or regions.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
