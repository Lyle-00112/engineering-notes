# Reconciling Logistics Subdomain Drift During Domain Verification (Webhook or Polling)

Choose webhook-triggered reconciliation when tenant onboarding needs a prompt completion signal; keep polling available to discover drift that arrives without a usable event. TL;DR: completion should mean that the published record matches the current tenant intent. A webhook should request that comparison, rather than declare the tenant ready.

For a logistics application assigning a subdomain to each tenant, the useful question is whether `harbor.dispatch.example` still points to the intended destination after an onboarding change. A successful write acknowledgment is too narrow a completion rule. So is an event that describes a superseded request. Ship the comparison first.

The data flow is small: save the desired tenant record with a revision, request publication, and let either a webhook or a scheduled poll trigger an observation of the published record. Compare that observation against the saved revision before updating readiness. This note uses a deterministic local example to make that rule inspectable; the publication and observation adapters are deliberately separate implementation work.

## Can a webhook replace polling for domain verification completion?

It should not. Consider an illustrative logistics tenant whose desired destination changes from `dispatch-a.example` to `dispatch-b.example` while onboarding is still pending. An event about the first request must not certify the second. The event can still be useful: it can wake a reconciler that reads the current intent and observes what is published now.

The following TypeScript example is runnable as a local simulation in a TypeScript-capable runtime. It makes no network requests and requires no service credentials. `readPublished` stands for an observation adapter returning a normalized CNAME target, without a trailing dot. The comparison contract intentionally covers one CNAME record; it is not a general DNS record comparator or a production worker.

```ts
type Intent = {
  tenant: string;
  host: string;
  target: string;
  revision: number;
};
type Readiness = { revision: number; ready: boolean };
type Observe = (host: string) => Promise<string | null>;

const intents = new Map<string, Intent>();
const readiness = new Map<string, Readiness>();
const published = new Map<string, string>();

function saveIntent(tenant: string, host: string, target: string): Intent {
  const next: Intent = {
    tenant,
    host,
    target,
    revision: (intents.get(tenant)?.revision ?? 0) + 1,
  };
  intents.set(tenant, next);
  readiness.set(tenant, { revision: next.revision, ready: false });
  return next;
}

const readPublished: Observe = async (host) => published.get(host) ?? null;

async function reconcile(tenant: string, observe: Observe): Promise<void> {
  const wanted = intents.get(tenant);
  if (!wanted) return;

  try {
    const actual = await observe(wanted.host);
    // A newer intent invalidates an observation already in flight.
    if (intents.get(tenant)?.revision !== wanted.revision) return;
    readiness.set(tenant, {
      revision: wanted.revision,
      ready: actual === wanted.target,
    });
  } catch {
    // An observation error cannot establish readiness.
    if (intents.get(tenant)?.revision === wanted.revision) {
      readiness.set(tenant, { revision: wanted.revision, ready: false });
    }
  }
}

async function demo(): Promise<void> {
  const tenant = 'harbor';
  const host = 'harbor.dispatch.example';
  const first = saveIntent(tenant, host, 'dispatch-a.example');
  published.set(host, first.target);

  const second = saveIntent(tenant, host, 'dispatch-b.example');
  // An old webhook wakes reconciliation; it supplies no readiness value.
  await reconcile(tenant, readPublished);
  console.log(readiness.get(tenant)); // revision: 2, ready: false

  published.set(host, second.target);
  // A later poll uses exactly the same comparison.
  await reconcile(tenant, readPublished);
  console.log(readiness.get(tenant)); // revision: 2, ready: true
}

void demo();
```

The two printed results describe a designed test case, not a measured deployment. The first reconciliation observes a valid destination for revision 1, but revision 2 wants something else.

It stays pending.

That is the pitfall worth catching before adding delivery retries: a completion handler that copies an event's success flag into tenant state can skip the comparison entirely. In this design, events contain a reason to check, never authority to approve a new destination.

## Define the observation before choosing the trigger

The example hides the most consequential decision inside `observe`. Specify what that function proves. A publication service reporting its stored configuration, an authoritative DNS answer, and an answer from the resolver used by an application are different observation contracts. Choose which evidence makes this particular onboarding step complete, and name it in state. Avoid a single ambiguous `verified` field shared by all three.

For the initial logistics workflow, define a narrow gate: the configured tenant hostname has the expected CNAME target in the chosen DNS observation. Record the observation time and the intent revision beside that result. Treat transport errors as unknown evidence and a different target as a mismatch; both can block activation, but they need different diagnostics. The small example collapses those outcomes into `ready: false` to keep its state transition visible. A deployed implementation should retain the distinction.

There is a standards boundary here. RFC 7489 defines DMARC policy discovery through DNS TXT records and email authentication alignment. It does not define a generic tenant-subdomain verification webhook or an onboarding completion event. Its DNS lookup and error-handling rules concern DMARC, so they cannot establish that a logistics application has finished provisioning a hostname.

Keep the claims narrow.

If the tenant subdomain will serve a tracking portal or an LLM shipment assistant, use separate checks for application routing and any required secure connection. The CNAME comparison above establishes only its declared DNS condition. It does not test the portal, authenticate a tenant, or authorize access to shipment data. Those gates belong in the application workflow, with their own evidence.

## Spend checks on uncertainty

For event-driven SaaS onboarding, use webhooks to request early observations when the publishing integration provides suitable events. Keep a polling path for pending tenants and a slower review of tenants already marked ready. Both paths should call the same reconciler, so changing the wake-up mechanism cannot change the definition of completion. Without a usable webhook contract, start with polling and preserve that interface. This is a trade-off between response time, observation traffic, and implementation effort. An illustrative 30-second polling interval schedules up to 120 observations per continuously pending tenant per hour, before retries. That arithmetic is a planning input, not a propagation promise or an API allowance. Set the interval using the onboarding delay the application can tolerate and the observation budget it can actually support. Bound retries, spread scheduled work, and stop rapid checks after an explicit onboarding deadline; let an operator or a later state change reopen the attempt. In a Node.js service, keep that scheduling policy outside the request handler so a tenant's browser session doesn't determine whether reconciliation continues.

Pending work survives the browser.

For a solo builder shipping an LLM feature, this control loop shouldn't need model calls. The desired target and observed target are structured values. Compare them directly. Reserve tokens for the shipment explanation or exception summary that users requested, and keep DNS reconciliation outside that request path. The code's observation interface also keeps the readiness rule independent of a particular publishing integration.

A webhook path still needs an explicit delivery contract. Verify the sender using the integration's documented mechanism, resolve its event identifier to an internal tenant, and enqueue a reconciliation request. Do not let an event-supplied tenant identifier alone authorize a configuration change. Repeated wake-ups should be safe; coalescing queued checks can reduce unnecessary observations. If events offer no useful delivery or authentication contract, their apparent speed is insufficient reason to rely on them.

There is another race beyond intent revisions. Two observations of the same revision can finish out of order. The local simulation invokes them sequentially; a deployed worker should serialize reconciliation per tenant, or enforce an equivalent ordering rule when committing observations. A revision check alone does not solve that case. With a database, make the revision comparison and readiness update conditional in one transaction rather than copying the in-memory pattern across processes.

## Before allowing the next tenant through

Exercise the stale-event sequence from the example, then test an intent change while observation is in flight, a missing record, an observation failure, and a repeated wake-up. Also test a target drifting after readiness was granted. Verify that the periodic path detects that mismatch and that the application follows an explicit policy for an existing tenant; do not let a transient observation error silently become a permanent tenant deletion. These tests protect the completion rule rather than merely checking that a handler ran.

At deployment, persist intent and publication work together or provide a recovery sweep that can find an intent whose publication request was never dispatched. Store the latest observation outcome, revision, and timestamp. Measure pending age, mismatches, observation errors, and time since the last completed reconciliation. A webhook delivery count alone cannot tell you whether `harbor.dispatch.example` matches its current target.

Start with the smallest loop that can explain a pending tenant. Add an event trigger when its contract improves completion time, and retain scheduled reconciliation wherever drift must remain detectable. For logistics onboarding, the durable decision is to gate readiness on current intent plus explicit evidence; webhook delivery and polling are ways to obtain that evidence.

## References

- RFC 7489, especially Section 6.6 on DMARC policy discovery and Section 6.6.3 on DNS query errors. Used here to bound what DMARC does and does not establish: https://datatracker.ietf.org/doc/html/rfc7489
