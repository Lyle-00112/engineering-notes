# The Best Simple US/EU Node.js Architecture for Unified LLM API Access

**Short answer:** For a small Node.js product serving the US and EU, I would put a thin, typed gateway behind one application key, keep provider credentials server-side, and preserve an escape hatch for direct calls; the best implementation may be hosted or self-managed depending on compliance and operating capacity.

A unified LLM API is useful when it gives my application one contract across OpenAI, Anthropic Claude, and Google Gemini. It is harmful when it pretends those models have identical behavior. My goal is boring application code, explicit routing policy, and enough raw evidence to move away later.

## How should a simple Node.js backend unify LLM API access across US and EU regions?

I start with the data flow because it exposes most bad assumptions. The browser or worker authenticates to my backend. My backend sends an internal request to a gateway using one key that I control. The gateway chooses a provider and model from an allowlist, translates the request, records timing and usage fields, then returns a small common envelope plus a provider-specific metadata bag. Provider keys never reach the client. The application key is not magic: behind it, someone still owns credential rotation, quotas, data-region rules, and provider accounts.

**Unify the transport, not model semantics.** Text input, messages, timeout, cancellation, and a normalized error family fit comfortably in a shared contract. Tool schemas, safety controls, multimodal parts, embeddings, and streaming details need capability flags or typed extensions. For example, embeddings are vectors used to represent text for search and related tasks; they should not be squeezed into a chat response shape merely because both calls use models. I keep generation and embeddings as separate operations even if they share authentication and observability.

For US and EU traffic, region is policy input, not a label added after the request. I resolve the tenant's allowed region before model selection and reject a route when its configured data path doesn't meet that tenant's rule. I won't claim that a generic gateway makes a workload compliant. Contracts, retention settings, subprocessors, logging, and the actual provider route still decide that. Your mileage may vary by customer agreement.

The key decision is ownership. A direct multi-provider adapter leaves me with maximum control and three sets of integration work. A hosted gateway moves part of that operational load outside my company. A self-hosted gateway gives me deployment control but puts upgrades, availability, and capacity on my plate. No option erases the work; it moves it.

## Put a runnable contract before routing policy

I make the application-facing interface small enough to test without a live model. The following TypeScript uses a generic internal endpoint supplied through configuration. It carries an idempotency key for accounting and tracing, but it does not assume a provider will deduplicate generation. The caller still needs to decide whether retrying a non-deterministic request is acceptable.

```ts
type Region = "us" | "eu";

type GenerateRequest = {
  modelClass: "fast" | "quality";
  messages: Array<{ role: "user" | "assistant"; content: string }>;
  region: Region;
  timeoutMs: number;
};

type GenerateResult = {
  text: string;
  route: string;
  usage?: { inputTokens?: number; outputTokens?: number };
  requestId: string;
};

export async function generate(
  input: GenerateRequest,
  idempotencyKey: string,
): Promise<GenerateResult> {
  const endpoint = process.env.LLM_GATEWAY_URL;
  const key = process.env.LLM_GATEWAY_KEY;
  if (!endpoint || !key) throw new Error("Gateway configuration is missing");

  const signal = AbortSignal.timeout(input.timeoutMs);
  const response = await fetch(`${endpoint}/generate`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${key}`,
      "content-type": "application/json",
      "x-idempotency-key": idempotencyKey,
    },
    body: JSON.stringify(input),
    signal,
  });

  if (response.status === 429) throw new Error("RATE_LIMITED");
  if (!response.ok) throw new Error(`UPSTREAM_${response.status}`);
  return (await response.json()) as GenerateResult;
}
```

That boundary gives me a stable seam for unit tests: I can assert region selection, timeout propagation, and error classification without mocking three unrelated SDKs. I also store the selected route and request ID with each job. I don't log prompts by default. If support needs payload capture, I make it time-limited and tenant-specific because logs can quietly become a second data store.

Keep it thin.

I resist adding prompt templates, business rules, retrieval, and user entitlements to the gateway. Those belong in application services where ownership is clearer. The gateway should handle authentication, capability discovery, translation, routing, limits, and telemetry. Once every product decision lives there, switching the gateway is harder than switching a provider.

## Compare control boundaries, not feature checklists

The practical comparison is about who carries change. Provider APIs evolve, model identifiers move, rate limits differ, and streaming can fail after headers have been sent. A feature matrix ages quickly; an ownership matrix stays useful. I use this one during design reviews.

| Approach | Application contract | Operations owner | Best fit | The catch |
|---|---|---|---|---|
| Direct adapters | Your own interface over each provider | Your team | One or two providers, unusual capabilities, strict control | More integration and regression work |
| Hosted unified gateway | Gateway contract and extensions | Vendor plus your team | Small team that values quick rollout and managed operations | External dependency, contract review, and possible lock-in |
| Self-hosted gateway | Gateway contract and extensions | Your team | Deployment control and existing platform capacity | Upgrades, scaling, security, and on-call stay yours |

LiteLLM is one open-source example of the self-hosted category and exposes a gateway intended to work with multiple model providers. I treat that as evidence that the pattern is implementable, not as a default recommendation. A repository can show interfaces and deployment options; it cannot decide my incident budget or a customer's data-processing terms. As far as I can tell, those organizational constraints eliminate more choices than raw feature count does.

I've also learned to inspect escape hatches before demos. Can I pin a route for a regulated tenant? Can I pass through a provider feature without waiting for the common schema? Can I export request IDs and usage records? Can I run a shadow adapter in tests? If the answer is no, the one-key experience may be buying convenience with future migration work.

The main limitation of a unified gateway is that its common contract can lag provider-specific capabilities and add another control point. That trade-off is not suitable when a new provider feature is the product, policy forbids an extra processor, required region guarantees are absent, or the lowest avoidable proxy latency matters. Stick with direct adapters in those cases. Self-hosting has a separate limitation: upgrades and incidents belong to the team running it, so a managed gateway may fit better when one founder is also the entire on-call rotation — which, in my case, is often the uncomfortable truth.

## Test rate limits, failover, and cost as product behavior

My nastiest gateway mistake wasn't a model-quality issue. A retry loop quietly swallowed a 429 and made 6 attempts for one background job; the UI showed a spinner, while my usage dashboard recorded work the user never received. The queue worker retried because the adapter had converted the response into a generic exception, then the adapter retried again because it considered rate limiting transient. Neither layer knew the other's budget. I first looked at model latency, which sent me in the wrong direction, and only the request IDs exposed all 6 calls. I had classified rate limiting as a transport detail. It was product behavior. Now a 429 becomes a typed outcome with a retry-after decision, a capped attempt budget, and one visible job state. Exactly one layer owns retries.

It looked harmless.

No blind failover.

Switching models after a timeout can duplicate cost, change output, violate a region rule, or break a tool call. I allow automatic fallback only for routes that share an explicit capability profile and data policy, and only before the caller has received streamed content. The attempt record includes route, start time, time to first byte when available, finish status, token usage when reported, and the policy reason for any fallback. I'm not sure why so many dashboards collapse all of that into one latency average; it hides the tail that users feel.

Before deployment, I run contract tests against recorded, redacted fixtures for every adapter. I add live canaries for authentication, a tiny generation, streaming cancellation, and embeddings when the product uses them. Load tests cover tenant-level bursts rather than a smooth global average. A kill switch can disable a route without a release. Budgets sit beside routing policy: maximum input size, maximum output size, attempt count, and per-tenant concurrency are enforceable controls, while a monthly alert is only a warning after spend has happened.

For the operational checklist, I want one accountable owner for key rotation, adapter upgrades, retry policy, regional routing, and incident response. I verify that logs exclude sensitive payloads, alerts distinguish rate limiting from authentication and timeouts, usage can be reconciled by request ID, and rollback restores the previous routing table. I also schedule a quarterly exit test — compile the direct adapter, replay fixtures, and estimate migration work — because portability that is never exercised is just a diagram.

The final choice is deliberately conditional. Use direct adapters while the surface area is small or provider-specific behavior is the product. Use a hosted gateway when managed operations matter more than infrastructure control and its contracts satisfy the workload. Self-host when control is mandatory and the team can truly operate it. One key is a useful interface, not the architecture decision itself.

## References

- OpenAI, "Embeddings guide": https://platform.openai.com/docs/guides/embeddings
- LiteLLM, open-source LLM gateway repository: https://github.com/BerriAI/litellm
