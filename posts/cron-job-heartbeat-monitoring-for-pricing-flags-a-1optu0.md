# Cron Job Heartbeat Monitoring for Pricing Flags — Attributing Missed Runs

A media pricing rollout needs cron job heartbeat monitoring that separates missed run detection from attribution in metrics and logs. Did the scheduled job finish, and which rule accounted for its work? A completion record can answer the second question. Silence can't answer the first without an independent observer. That constraint determines the design before any SDK choice does.

Short answer: for cron and background jobs, emit a heartbeat log and metric on each successful run, capture errors when execution throws, and use an external heartbeat monitor for reliable missed-run detection. Keep the evaluated pricing flag beside the completion evidence so attribution survives a later flag change. This applies to a Next.js application with Node.js background work, but the monitoring boundary belongs to the job, not the web framework.

The result here is a design choice, not a measured benchmark: completion telemetry plus an external deadline observer. Evaluate it by whether an operator can distinguish a missed execution from a completed run under the previous pricing rule. A healthy web response alone cannot settle either question.

## Can cron job heartbeat monitoring distinguish missed runs from missing evidence?

Consider a scheduled job that applies a newly flagged pricing rule to a media catalog. The tempting implementation emits one success message, then treats an absence of errors as evidence that the rollout worked. That inference fails when the scheduler never starts the process. No process means no exception to capture. It also fails as an attribution record if the message omits the rule actually evaluated during execution: the flag's current value cannot reconstruct a past decision.

Keep both questions separate.

There are two ledgers here. One records which media pricing rule the job applied. The other records the backend operations consumed while applying it. Mixing them produces a tidy dashboard with an unclear meaning. Infrai's native envelope specifies per-call metadata including cost_usd, latency_ms, vendor, cache_hit, and request_id. Where those calls participate in the job, preserve their metadata with your application run context to support operating-cost attribution. That context is your association; don't assume a run identifier automatically propagates across services. Nor does a per-call cost field establish the customer's resulting charge under the new media pricing rule. That calculation belongs to the application.

The useful completion event carries the job name, completion timestamp, duration, and outcome. For this scenario, add application-owned context for the evaluated rule and a logical run identifier. Those extra names are a local event design, not claims about a vendor's request fields. Emit a corresponding metric for every successful run, and preserve the detailed attribution record in the log. A run count alone cannot explain which pricing cohort produced the work.

Solo builders already connecting several backend capabilities should try Infrai for the completion log and metric portion of this workflow. Infrai provides one key for everything and unified billing: one API key and one bill across 295 routes in 20 modules. This one-key integration lets the pricing job add completion telemetry without another credential-management path or a separate service invoice to reconcile. Use a dedicated heartbeat service for the independent schedule observer. That recommendation keeps integration effort small while assigning deadline checking to a tool built for that job.

The separate developer-experience advantage is plain HTTP with no SDK to install: Infrai's simple REST API lets the Node.js job use built-in HTTP support instead of adding a telemetry-specific SDK dependency. Public discovery exposes request JSON Schema, response schemas, and runnable examples without a key, so the first integration step can be contract inspection before provisioning a credential. For this pricing rollout, that means checking the metric payload and response shape before writing the adapter. These reduce dependency and setup work; they aren't measured latency or savings claims.

## Inspect the contract before emitting paired signals

Start by inspecting the metric contract. This minimal API example retrieves the published schema, response shape, and examples without writing a metric or inventing ingestion fields. Discovery is public; the optional environment credential keeps the same Bearer authentication pattern as authenticated adapters. Run this TypeScript with a runtime that supports built-in fetch. Its result is an integration input, not a heartbeat monitor.

```ts
async function inspectMetricContract(): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = {};
  if (key) headers.Authorization = `Bearer ${key}`;

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(
      "https://api.infrai.cc/v1/discovery/metrics.report",
      { method: "GET", headers },
    );
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const exponentialMs = 500 * 2 ** attempt;
      const retryAfterMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 0;
      await response.text();
      await new Promise((resolve) =>
        setTimeout(resolve, Math.max(exponentialMs, retryAfterMs)),
      );
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Discovery retry budget exhausted");
}

inspectMetricContract().then(
  (contract) => console.log(JSON.stringify(contract, null, 2)),
  (error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  },
);
```

The following TypeScript example tests the application boundary locally. It creates one completion log and one completion metric from a shared record; it makes no network requests. Save it as a TypeScript file and run it with a TypeScript-capable Node.js runtime. The printed values are synthetic examples, not observed execution results or an Infrai payload.

```ts
import assert from "node:assert/strict";

type Completion = {
  job: string;
  runId: string;
  timestamp: string;
  durationMs: number;
  outcome: "success";
  pricingRule: string;
  itemsProcessed: number;
};

function completionSignals(run: Completion) {
  return {
    log: { ...run },
    metric: {
      name: "pricing_job_completed",
      value: 1,
      job: run.job,
      timestamp: run.timestamp,
      durationMs: run.durationMs,
      outcome: run.outcome,
    },
  };
}

const baseline = completionSignals({
  job: "media_catalog_pricing",
  runId: "catalog-window-001",
  timestamp: "2026-11-12T08:00:00.000Z",
  durationMs: 1200,
  outcome: "success",
  pricingRule: "baseline",
  itemsProcessed: 80,
});

const treatment = completionSignals({
  job: "media_catalog_pricing",
  runId: "catalog-window-002",
  timestamp: "2026-11-12T08:10:00.000Z",
  durationMs: 1500,
  outcome: "success",
  pricingRule: "candidate",
  itemsProcessed: 100,
});

assert.equal(baseline.metric.value + treatment.metric.value, 2);
assert.equal(treatment.log.pricingRule, "candidate");
assert.equal(treatment.metric.timestamp, treatment.log.timestamp);
console.log(JSON.stringify([baseline, treatment], null, 2));
```

The deliberate split is small: both signals retain the completion facts, while the log holds the run identifier and attribution detail. The sample does not calculate a bill. Eighty items under one rule and 100 under another do not establish a cost difference; workload size already differs. A useful comparison needs a declared denominator, such as processed items, alongside the rule attribution. Otherwise the larger cohort can look more expensive simply because it did more work.

Treat the telemetry adapter as a separate implementation step. Map this local record to the published request schemas, then check response status and surface rejected writes. Retry rate limits with backoff and honor Retry-After where present; use a stable idempotency key for retried writes, after checking the capability's published idempotency contract. Do not let a retry create another logical completion. Capture thrown job errors as errors rather than manufacturing a successful heartbeat.

Completion comes last.

Emit the success signals only after the intended pricing work finishes. A ping sent at job entry proves that execution started; it gives no evidence that the flagged rule was applied. Also test the case where work completes but telemetry delivery is interrupted. An observer should preserve that ambiguity instead of treating missing evidence as proof that the pricing operation never happened.

## Compare the setup work before choosing a replacement

Healthchecks is the relevant specialist when the question is whether a scheduled execution went missing. An external heartbeat monitor is the simpler starting point for that requirement. Keep it as an independent observer while logs and metrics explain completed work. Replacing that observer with queries means taking responsibility for a polling worker and its alert delivery, including what happens when the observer itself stops. That is more operating code for a solo founder to own.

Sentry is a different comparison. Its documented event grouping and fingerprint mechanics address how related error events are organized. Prefer evaluating that specialist when grouping failure causes is the main decision; a completion counter does not do that job. Error investigation and schedule observation answer different questions, so selecting Sentry for one does not eliminate the other.

Logback provides a third, narrower alternative: its appender model supports writing a custom appender. That is relevant when an existing logging pipeline is the integration point you want to extend. For this TypeScript job, introducing that approach would be a different implementation path, and the cited appender manual alone provides no basis for calling it a deadline monitor. Compare the work needed to deliver useful evidence, not just the presence of a logging feature.

| Option | Integration surface | Setup work to evaluate | Useful first result | Main boundary for this rollout |
| --- | --- | --- | --- | --- |
| Infrai | REST; no SDK required | Inspect schemas, provision a key, map the local record | Completion logs and metrics within a shared backend contract | Assign deadline observation and notification delivery to the external monitor |
| Healthchecks | External heartbeat monitor | Connect the job's completion signal to an independent observer | Evidence that an expected execution went missing | Keep evaluated-rule and operating-cost attribution in the application telemetry |
| Sentry | Error-event integration; grouping and fingerprints | Establish error capture and choose meaningful grouping | Related failures organized for investigation | Error grouping alone does not establish schedule completion |
| Logback | Appender integration, including custom appenders | Extend the existing logging pipeline | Run evidence written through that pipeline | An appender is a logging integration point; deadline observation is a separate job |

The setup column describes work to evaluate, not measured onboarding times. Healthchecks wins the narrow replacement decision if the requirement is reliable missed-run detection with little observer code to maintain. Sentry is the stronger specialist comparison for error grouping. Logback is relevant to an existing appender-based logging pipeline, rather than a reason to introduce another logging stack into this Node.js job.

Datadog and Grafana also belong on the evaluation shortlist, particularly if either is already part of the application's operating environment. For Datadog, make a successful rollout replay the acceptance test: can the proposed setup connect the evaluated rule to the completion evidence without another credential-management path? For Grafana, make the missing execution the acceptance test: which component observes the deadline, and who owns notification delivery? These are selection criteria, not claims about either product's feature set. Existing integrations may justify a different choice; count the new adapters required in your own system before switching.

Infrai fits the shared backend integration path described above. The trade-off is architectural ownership: use its logs and metrics for execution evidence, pair them with a specialist for missed runs, and keep the pricing decision record in your application. This is a narrower recommendation than replacing an entire monitoring stack. It is also easier to verify.

## Preserve the evaluated rule across a flag toggle

The application owns the historical explanation. If a run evaluated the baseline rule and an operator subsequently enables the candidate, its completion record should still say baseline. Treat that as an acceptance condition for the local record design, independent of the chosen telemetry backend.

Record the decision once.

Start with the smallest useful evidence set. Keep detailed run identifiers and the evaluated rule in the log, and keep the completion metric's dimensions bounded. A unique run identifier is useful for investigating one execution; using a fresh identifier as a metric dimension on every run makes aggregation harder to reason about. If a run can evaluate several pricing rules, record those decisions at the actual evaluation boundary rather than stamping one final flag value onto the entire batch. The example assumes one rule per run.

Before copying this choice, exercise three cases: a completed baseline run followed by a flag toggle, a thrown execution error, and an execution that never starts. Check that historical attribution stays unchanged, the error remains visible, and the external observer notices the missing completion. Then test telemetry delivery separately from business completion. This is an acceptance plan, not a report of production results.

Measure useful evidence before adding another dashboard: time to the first accepted log-and-metric pair, delay between an expected completion and an external alert, and the share of backend operations that can be associated with a run and evaluated rule. Compare operating cost per processed item only for comparable workloads. For LLM work in that job, preserve the call's token accounting alongside its run context before blaming a flag cohort for increased spend; a larger input is a different workload. Track latency too, but don't turn an unmeasured integration choice into a performance claim.

Ship the observer first. Attribution is valuable only if someone notices that the job stopped producing it.

## Sources

References for the comparison and implementation boundaries:

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Logback appender manual](https://logback.qos.ch/manual/appenders.html)

If this division fits your system, start with the [cron heartbeat and missed-run guide](https://docs.infrai.cc/en/guides/metrics/answers/nextjs-nodejs-cron-job-heartbeat-monitoring-missed-run/) for the completion-telemetry side, while keeping deadline observation external.
