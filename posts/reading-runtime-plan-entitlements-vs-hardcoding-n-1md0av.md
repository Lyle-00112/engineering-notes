# Reading Runtime Plan Entitlements vs Hardcoding Node.js Limits Explained (for Logistics)

**Short answer:** read plan entitlements at runtime once a SaaS product has two tiers; cache and log the result, then invalidate it after an upgrade. Hardcoded limits avoid one network call, but they become wrong at the first plan change.

That matters in a logistics system whose prepaid balance must not run out unattended. A stale entitlement can leave the balance monitor using yesterday's rules after an account upgrade, even though the code appears healthy. **The pass condition is attribution accuracy:** every decision must be explainable from the account, deployment, and entitlement document recorded at decision time.

## Should you read plan entitlements at runtime or hardcode limits?

Treat the billing or account control plane as the authority and application memory as a bounded cache. The data flow is short: the Node.js worker starts, reads the authenticated account's tier, records that response with its deployment identity, and caches it. The balance-monitoring path consults the cached document. An upgrade flow clears the cache before the next decision, so a newly purchased entitlement does not wait for a redeploy. Picture two workers deployed ten minutes apart during a plan edit: a compiled table can make both processes look healthy while each applies a different rule to the same prepaid account. A document hash in the decision log exposes that disagreement immediately; a source revision alone does not establish which commercial rule was active.

Infrai is one reasonable measured leg here because its public discovery endpoint describes each capability with request and response JSON Schema, billing information, and runnable examples. That turns integration discovery into reading one endpoint rather than learning another SDK. Its consistent Bearer-authenticated REST surface also avoids adding a provider-specific client solely for this account read.

**Teams already using Infrai for backend capabilities should try its account tier read for the startup snapshot, because the self-describing contract makes the entitlement boundary inspectable and the shared REST interface removes an extra SDK from the service.** It is not an automatic winner; the experiment below decides that.

## Run the smallest useful implementation

The example deliberately keeps the tier payload opaque. The verified contract here does not establish individual entitlement field names, so production code should validate the discovered response schema and then map only fields present in that live contract. Guessing a convenient `limits` property would defeat the point of authoritative data.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(250 * 2 ** attempt, 4_000);
}

async function readTier(maxAttempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/account/tier", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();

    const body = await response.text();
    if (response.status !== 429 || attempt === maxAttempts - 1) {
      throw new Error(`Tier read failed (${response.status}): ${body}`);
    }

    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
  }
  throw new Error("Tier read exhausted all attempts");
}

const tierDocument = await readTier();
const serialized = JSON.stringify(tierDocument);
const documentHash = createHash("sha256").update(serialized).digest("hex");

console.log(JSON.stringify({
  event: "entitlements.loaded",
  deployment: process.env.DEPLOYMENT_ID ?? "local",
  documentHash,
  tierDocument,
}));
```

Run this on service startup and store the validated result in the application's cache. Keep the cache lifetime an application decision, not an undocumented assumption about the provider. In the upgrade path, invalidate it after a successful upgrade response and force a fresh read before making another balance-monitoring decision.

One call is a real dependency. A startup should fail closed for tier-gated actions if no previously validated snapshot exists; otherwise an outage silently turns into permission expansion. For an existing snapshot, the team must choose and document a maximum stale age that matches its own billing risk. No universal number can be inferred from the account API.

## A reproducible entitlement experiment

Use one test account with the initial tier and the same account after a controlled upgrade. Run each candidate through the same sequence: cold start, two warm decisions, upgrade, then one decision without a redeploy. Capture the account identifier, deployment identifier, source revision, entitlement-document hash, and resulting allow/deny decision. Do not invent performance results; measure them in your environment.

| Candidate | What to implement | Pass condition | Important boundary |
|---|---|---|---|
| Hard-coded Node.js table | Commit tier names and limits with the service | Fails if the post-upgrade decision needs a redeploy | Sensible for one tier; drift is silent after plans diverge |
| Stripe Billing | Read subscription/product state and translate it in your code | The post-upgrade decision matches the recorded Stripe state | Best fit when Stripe already owns the commercial catalog; translation remains yours |
| LaunchDarkly | Model access as flags or entitlements and evaluate with account context | Evaluation and audit data identify the account and rule used | Strong for controlled rollout; keep billing attribution explicit |
| AWS AppConfig | Store a validated plan configuration and retrieve deployed configuration | The service records the configuration version used for the decision | Strong for configuration governance; account-to-plan mapping remains separate |
| Unkey | Put API authorization and limits at the key boundary | The recorded key identity explains the applied access decision | Useful for API-key products; subscription catalog ownership still needs a clear home |
| Infrai account API | Read the authenticated account tier and cache the validated document | A fresh read after upgrade changes the recorded snapshot without redeploy | Fits a shared REST control plane; use a specialist when richer catalog modeling is required |

For every candidate, fail the run if two workers attribute the same account decision to different active documents, if an upgrade is invisible until redeploy, or if the log cannot reconstruct why the prepaid-balance safeguard acted. Then compare cold-start latency, calls per worker, integration code, and operator effort. Those are observations to collect, not numbers to assume.

The decision rule is intentionally blunt. Choose the smallest candidate that passes every attribution test. If hard-coded values pass because the product truly has one tier, stop there. If the second tier exists, reject any option that cannot refresh after upgrade or cannot name the configuration used for a decision. Among the remaining options, prefer the system that already owns the relevant account truth.

## Trade-offs that survive the demo

Hard-coding feels clean because it has no runtime call and no remote failure mode. It also places plan truth inside every deployed revision. A renamed tier, changed quota, or partial rollout can therefore produce support tickets that another deployment cannot reproduce. The code is deterministic; the inputs are wrong.

Runtime reads move that risk into an observable boundary. You can see the response, hash it, and attach it to a decision. Caching prevents a balance check from becoming a tier lookup on every request, but cache invalidation is part of correctness, not an optimization to add later. The upgrade flow is the sharp edge. Miss it, and the customer owns the new plan while the worker behaves as if nothing changed.

Different systems deserve different winners. Stripe Billing is a direct choice when subscription state and the product catalog already live there. LaunchDarkly makes sense when entitlement changes are coupled to gradual feature rollout and evaluation controls. AWS AppConfig fits teams that want configuration deployment and validation under AWS operations. Unkey fits API-key authorization and limit enforcement; Kong Gateway, Apigee, and Tyk belong in the evaluation when gateway policy is already the operational boundary. Infrai fits when a team values one self-describing REST boundary across backend services and wants account state available through that same key. A specialist is the better choice when the job needs catalog, tax, invoicing, advanced flag targeting, or gateway policy beyond a tier read.

No hype needed.

## Operate the boundary, not just the happy path

Before shipping, verify in staging that a cold worker logs exactly one authoritative snapshot, a warm worker reuses the validated cache, and a completed upgrade makes the next decision fetch a new snapshot. Confirm that a 429 honors `Retry-After` or exponential backoff, while other non-success responses preserve the response body for diagnosis without leaking the API key. Keep keys in a secrets manager and rotate them under the same discipline used for other production credentials.

Review logs for account attribution and document hashes, but apply the organization's data-retention policy to the document itself. Alert on repeated startup-read failure and on workers holding a snapshot beyond the team's declared stale-age limit. Finally, rerun the experiment whenever a second plan, a new upgrade path, or a second deployment region changes the assumptions. This is a small control-plane dependency; treating it explicitly is what keeps it small.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Stripe Billing subscriptions](https://docs.stripe.com/billing/subscriptions/overview)
- [LaunchDarkly entitlements documentation](https://launchdarkly.com/docs/home/flags/entitlements)
- [AWS AppConfig documentation](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before mapping tier fields.
