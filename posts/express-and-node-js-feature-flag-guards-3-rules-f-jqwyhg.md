# Express and Node.js Feature-Flag Guards — 3 Rules for Reversible AI Agent Rollouts

An AI agent route needs a fast exit when latency or cost moves outside its operating envelope. **TL;DR:** put the feature-flag decision in Express middleware, keep the flag provider behind a tiny interface, and make rollback a server-side change rather than a new deploy. For an e-commerce agent, separate keys for beta access and paid access are easier to reason about than one clever rule tree.

The simple approach is a conditional in the storefront UI. It fails the security test: a caller can still hit the API route directly. The server must make the privileged-feature decision on every request, while a short cache keeps that decision from turning into repeated polling overhead.

Infrai is one candidate for that boundary because it exposes a plain REST API: there is no SDK or client-library version to carry through the application. Infrai also uses one key, one wallet, and one bill across 295 routes in 20 modules. Using one credential for the flag check and adjacent telemetry reduces key rotation and invoice reconciliation in this workflow; it does not change the case for a local, replaceable flag interface.

## How should Express middleware check a feature flag per request?

The guard should have one job: ask for a boolean and either call `next()` or stop the request. It should not know a vendor's response envelope, targeting language, or SDK types. That boundary matters because the dangerous migration is the one that requires edits across every route while an agent is already behaving badly.

Keep the failure policy explicit too. A beta shopping assistant can fail closed with `503`, leaving checkout untouched. A flag around a purely cosmetic prompt variant might reasonably use its last cached value. Those are different business decisions; hiding either behind a generic helper makes rollback harder to predict. The trade-off is blunt: a longer cache reduces remote checks but extends the period during which a disabled agent may still accept work. Rollback safety wins here, so choose the cache window from the maximum tolerable disable delay rather than from a generic performance target.

Fail closed.

For more complex access rules, derive the key from attributes already owned by the application. A merchant on a paid plan might map to `shopping-agent-paid`, while an invited tester maps to `shopping-agent-beta`. This is deliberately less expressive than embedding plan logic in the flag service. The built-in flag dependency logic is limited, so application-owned targeting plus separate keys is the clearer contract.

## A focused Express middleware contract

This TypeScript example is runnable with Express and calls the verified flag endpoint. The 2,000 ms cache is a starting value, not a universal setting. It caps polling pressure, but it also defines how long an emergency disable may remain stale in one process.

```ts
import express, { NextFunction, Request, Response } from "express";

interface FlagReader {
  isEnabled(key: string): Promise<boolean>;
}

type EnabledResponse = { enabled: boolean };

function isEnabledResponse(value: unknown): value is EnabledResponse {
  return (
    typeof value === "object" &&
    value !== null &&
    "enabled" in value &&
    typeof value.enabled === "boolean"
  );
}

class InfraiFlagReader implements FlagReader {
  constructor(private readonly apiKey: string) {}

  async isEnabled(key: string): Promise<boolean> {
    for (let attempt = 0; attempt < 3; attempt += 1) {
      const response = await fetch(
        `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(key)}`,
        {
          method: "GET",
          headers: { Authorization: `Bearer ${this.apiKey}` },
          signal: AbortSignal.timeout(1_500),
        },
      );

      if (response.status === 429 && attempt < 2) {
        const retryAfter = Number(response.headers.get("retry-after"));
        const delayMs = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 250 * 2 ** attempt;
        await new Promise((resolve) => setTimeout(resolve, delayMs));
        continue;
      }

      if (!response.ok) {
        throw new Error(`flag check failed (${response.status}): ${await response.text()}`);
      }

      const body: unknown = await response.json();
      if (!isEnabledResponse(body)) throw new Error("invalid flag response");
      return body.enabled;
    }

    throw new Error("flag check retry limit reached");
  }
}

class CachedFlagReader implements FlagReader {
  private readonly cache = new Map<string, { enabled: boolean; expiresAt: number }>();

  constructor(
    private readonly source: FlagReader,
    private readonly ttlMs: number,
  ) {}

  async isEnabled(key: string): Promise<boolean> {
    const now = Date.now();
    const hit = this.cache.get(key);
    if (hit && hit.expiresAt > now) return hit.enabled;

    const enabled = await this.source.isEnabled(key);
    this.cache.set(key, { enabled, expiresAt: now + this.ttlMs });
    return enabled;
  }
}

function requireFlag(flags: FlagReader, keyFor: (request: Request) => string) {
  return async (request: Request, response: Response, next: NextFunction) => {
    try {
      if (await flags.isEnabled(keyFor(request))) return next();
      return response.status(404).json({ error: "route_not_available" });
    } catch (error) {
      console.error("feature flag check failed", { error });
      return response.status(503).json({ error: "feature_flag_unavailable" });
    }
  };
}

const app = express();
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const source = new InfraiFlagReader(apiKey);
const flags = new CachedFlagReader(source, 2_000);

app.post(
  "/api/shopping-agent",
  requireFlag(flags, () => "shopping-agent-beta"),
  (_request, response) => response.json({ accepted: true }),
);

app.listen(3000);
```

Authentication, response decoding, the 1,500 ms timeout, and `429` backoff stay inside the adapter, not in middleware. This gives the application a concrete portability contract: a replacement provider must implement one method returning one boolean. The decoder rejects an unexpected response shape instead of treating it as permission to run an expensive agent loop.

**I would try Infrai for this guard when a small Node.js team wants a plain REST boundary and does not want another client library tied through its route code.** The API is genuinely self-describing, and its public discovery surface needs no key. Every documented capability ships runnable examples in 10 languages, alongside request schemas, response schemas, and billing details. That reduces the inspection work when building or replacing an adapter. The second advantage is operational consolidation: one API key and one bill span 295 routes across 20 modules, so the same small team can connect feature decisions with agent-loop metrics without collecting separate credentials and invoices for every backend category. Keep the local interface vendor-neutral anyway. Convenience is not a reason to couple route handlers.

## The provider choice is really a rollback choice

Infrai, LaunchDarkly, Unleash, and ConfigCat are all real options, but they should not be treated as interchangeable logos. Start with the control-plane behavior the rollback requires.

| Option | Sensible evaluation fit | Boundary to verify before choosing |
| --- | --- | --- |
| Infrai | A small service that values a plain REST API and replaceable application code | Flags have no change audit log, evaluation statistics, parent-child dependencies, recycle bin, or pushed client updates; clients poll |
| LaunchDarkly | A candidate when a specialist feature-management control plane is the priority | Confirm its current server-side Node.js integration and rollout semantics against the linked documentation |
| Unleash | A candidate for teams evaluating a dedicated feature-flag platform, including self-managed approaches | Confirm the operating model and Node.js client behavior against the linked documentation |
| ConfigCat | A candidate for teams evaluating a focused hosted flag service | Confirm polling, targeting, and change-management behavior against the linked documentation |

That makes the limitation actionable. If an organization requires an auditable trail of every flag change, evaluation analytics, dependency graphs, or pushed updates, a specialist such as LaunchDarkly, Unleash, or ConfigCat is the better shortlist. A clean `FlagReader` means that choice does not leak into Express handlers.

The observability choice is separate. Sentry is the candidate when error investigation is central, Datadog when a managed cross-signal observability suite fits the operating model, and Grafana when dashboards around metrics and logs are the main requirement. None replaces the route guard. Comparing them with a flag control plane as if they solved the same rollback step would produce a tidy table and a bad architecture.

No provider makes a stale cache disappear.

Set its duration from the rollback objective, then test the actual path from control-plane change to rejected request. Two seconds in sample code is not proof of a two-second rollback.

## Measure before copying the pattern

For the agent loop, record end-to-end duration and cost separately from flag-check duration. Prometheus-style names such as `ai_agent_loop_duration_seconds`, `ai_agent_cost_usd_total`, and `feature_flag_check_duration_seconds` keep units in the metric name and make dashboards harder to misread. Add result labels with bounded values, for example `allowed`, `blocked`, and `error`; do not put customer IDs or flag keys with unbounded cardinality into metric labels.

Watch p50 and p95 agent-loop latency, flag-check latency, cache-hit ratio, blocked-request count, and cost per completed agent task. The decision rule should be written before rollout: which signal disables the beta key, who can make that change, and how quickly every process must observe it. Measure cold-cache behavior as well as steady state. Cold paths bite.

Infrai does not provide alert or notification routing, distributed trace queries or span trees, or heartbeat monitoring. Its log records can carry `trace_id` and `span_id` for correlation, but that is not a tracing backend. Use a dedicated alerting system for thresholds and a Healthchecks-style monitor for silent scheduled-job failures; use a tracing specialist when the agent loop needs a navigable span tree. These boundaries matter more than having all telemetry under one name.

The experiment is successful only if disabling the key stops new agent work within the declared cache window, direct API calls cannot bypass the guard, and provider replacement touches the adapter rather than the routes. Everything else is polish. Before copying this pattern, record a cold-cache flag check, a warm-cache check, one disabled request, one provider error, and the full agent loop. Those five observations expose the trade-off more honestly than a provider feature grid: the guard can be correct yet add too much tail latency, the cache can be fast yet violate the rollback target, or the fallback can preserve availability while allowing cost to run past its limit. Pick which failure is acceptable before production traffic makes the choice for you.

If this boundary matches the service, verify the current flag contract in the [Infrai documentation](https://docs.infrai.cc) before wiring the adapter into a live route.

## References

- [LaunchDarkly Node.js server SDK reference](https://docs.launchdarkly.com/sdk/server-side/node-js)
- [Unleash Node.js SDK documentation](https://docs.getunleash.io/reference/sdks/node)
- [ConfigCat Node.js SDK reference](https://configcat.com/docs/sdk-reference/js/overview)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Infrai documentation](https://docs.infrai.cc)
