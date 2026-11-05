# Healthtech Backend Metrics Dashboard: 3 Checks for Cron Jobs and API Failures

A scheduled healthtech import needs an alert when expected results stop arriving. Start with a heartbeat check for missed runs and a metrics dashboard for success counts, API failures, and business events. TL;DR: use metrics to explain what executed, a Healthchecks-style deadline to detect what never executed, and an explicit result count to catch jobs that finish without producing useful output. Keep cost attribution attached to the same run evidence.

A completed process can still deliver zero results. That distinction matters more than another chart. For a solo builder shipping an LLM-assisted import, the useful question is which scheduled delivery consumed resources, reached its destination, and produced accepted records. A model call succeeding is only one step along that path.

## How should a backend metrics dashboard distinguish cron jobs from useful imports?

Use three checks: arrival by a deadline, execution outcome, and accepted output. They answer different questions. A heartbeat establishes whether the expected execution checked in. Metrics describe counts, durations, backlog sizes, and API error trends. A business-event count records whether the import actually produced accepted results. Error records can add context to a failed execution, while the time-series chart remains in the metrics layer.

The data flow is small: the scheduler identifies an expected delivery, the import worker records its execution and any model-call metadata, and the completion step records accepted results. A separate heartbeat service watches the delivery deadline. The dashboard joins the evidence using an application-owned run identifier; a small evaluator decides whether missing output warrants an alert. Keep patient content out of this example: its inputs are synthetic operational counts and opaque run labels, with no claim that this alone establishes a compliance boundary.

Count the outputs. Process completion isn't delivery.

The next example makes that policy executable before choosing a metrics backend. Its types are an application contract, not a vendor request schema. The deadlines and counts are synthetic fixtures; they are neither measured performance nor product pricing. The rule separates an overdue run from a completed run with unexpectedly empty output, since the operator needs a different investigation for each.

## Implement the result check before wiring the dashboard

Save this as `import-evidence.ts` and run it with `npx tsx import-evidence.ts` in a project with TypeScript tooling and a runtime providing fetch. Set `METRICS_QUERY_URL` to the platform's verified metrics query URL and `INFRAI_API_KEY` to your key. The example prints the unmodified query response alongside three synthetic policy decisions; it does not assume a response field layout or add undocumented filters. The heartbeat service and the application's notification worker remain responsible for delivery of alerts; printing a policy decision does not send one.

```ts
type Delivery = {
  runId: string;
  dueAtMs: number;
  graceMs: number;
  minimumAccepted: number;
};

type Evidence = {
  runId: string;
  completedAtMs: number;
  accepted: number;
  apiFailures: number;
  modelCallCostsUsd: number[];
};

type Decision = {
  runId: string;
  status: "pending" | "missed" | "empty" | "partial" | "delivered";
  accepted: number | null;
  modelSpendUsd: number | null;
  modelSpendPerAcceptedUsd: number | null;
  apiFailures: number | null;
};

function evaluate(
  delivery: Delivery,
  evidence: Evidence | undefined,
  nowMs: number,
): Decision {
  if (evidence && evidence.runId !== delivery.runId) {
    throw new Error("Evidence belongs to a different delivery");
  }
  if (evidence && evidence.completedAtMs > nowMs) {
    throw new Error("Completion evidence is in the future");
  }
  const modelSpendUsd = evidence
    ? evidence.modelCallCostsUsd.reduce((total, cost) => total + cost, 0)
    : null;
  const status: Decision["status"] = !evidence
    ? nowMs > delivery.dueAtMs + delivery.graceMs ? "missed" : "pending"
    : evidence.accepted === 0 && delivery.minimumAccepted > 0
      ? "empty"
      : evidence.accepted < delivery.minimumAccepted ? "partial" : "delivered";
  return {
    runId: delivery.runId,
    status,
    accepted: evidence?.accepted ?? null,
    modelSpendUsd,
    modelSpendPerAcceptedUsd: evidence && evidence.accepted > 0
      ? modelSpendUsd! / evidence.accepted
      : null,
    apiFailures: evidence?.apiFailures ?? null,
  };
}

const nowMs = Date.parse("2026-11-02T08:20:00Z");
const deliveries: Delivery[] = [
  { runId: "roster-a", dueAtMs: nowMs - 1_200_000, graceMs: 600_000, minimumAccepted: 1 },
  { runId: "roster-b", dueAtMs: nowMs - 1_200_000, graceMs: 600_000, minimumAccepted: 1 },
  { runId: "roster-c", dueAtMs: nowMs - 1_200_000, graceMs: 600_000, minimumAccepted: 1 },
];
const completed: Evidence[] = [
  { runId: "roster-b", completedAtMs: nowMs - 60_000, accepted: 0, apiFailures: 2, modelCallCostsUsd: [0.02, 0.01] },
  { runId: "roster-c", completedAtMs: nowMs - 60_000, accepted: 40, apiFailures: 1, modelCallCostsUsd: [0.04] },
];
const byRun = new Map(completed.map((entry) => [entry.runId, entry]));
for (const delivery of deliveries) {
  console.log(JSON.stringify(evaluate(delivery, byRun.get(delivery.runId), nowMs)));
}

async function queryMetrics(): Promise<unknown> {
  const endpoint = process.env.METRICS_QUERY_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!endpoint || !apiKey) throw new Error("Set the metrics URL and API key");
  const url = new URL(endpoint);
  if (url.protocol !== "https:" || url.pathname !== "/v1/metrics/query" || url.search) {
    throw new Error("Use the HTTPS metrics query endpoint without filters");
  }
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      redirect: "error",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const retryAt = retryAfter === null ? NaN : Date.parse(retryAfter);
      const waitMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(retryAt) ? Math.max(0, retryAt - Date.now()) : 0;
      await new Promise((resolve) => setTimeout(resolve, Math.max(waitMs, 1000 * 2 ** attempt)));
      continue;
    }
    if (!response.ok) throw new Error(`Metrics query ${response.status}: ${body}`);
    return JSON.parse(body) as unknown;
  }
  throw new Error("Metrics query retry budget exhausted");
}

queryMetrics().then((result) => console.log(JSON.stringify(result))).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The fixtures produce a missed delivery, an empty delivery, and a delivered batch. The last batch has an API failure and still delivers 40 accepted records. That is intentional: an error count alone cannot establish that a delivery failed. A successful retry might leave an error event behind. The empty delivery retains its recorded spend, but its spend per accepted record is null. Dividing by zero or displaying zero would turn failed work into a reassuring financial signal.

This evaluator assumes one authoritative completion record per delivery. Before using it in a worker, persist that record with a unique run key and count distinct accepted outputs. Retries need separate attempt identifiers so their resource consumption can accumulate without doubling the delivered-record count. These are application design requirements, not guarantees supplied by a dashboard.

The example evaluates output after completion. For a worker that starts and then hangs, the heartbeat deadline must still expire. A missing completion record means missing completion evidence; it does not prove the process never started. Persist starts separately if operators need to distinguish those cases. The grace interval here is 10 minutes, chosen only to make the fixture readable. Set the real deadline from the feed's delivery contract.

## Attribute model spend to accepted work

For the synthetic empty batch, recorded model spend is 0.03 USD and accepted output is zero. That is a diagnostic input, not a claim about a vendor's unit price. A cost chart that groups only by model would miss the operational question: which expected delivery consumed those calls without producing a result? Keep the scheduled delivery as the accounting unit and retain attempt-level evidence for investigation. Aggregate the dashboard by import type or time window; keep individual run identifiers in the underlying evidence rather than making every run a permanent chart dimension.

Infrai offers a plain REST API with no SDK to install, covering 295 routes across 20 modules under one key and one bill. Anything that can send an HTTP request can call it. Using model calls and metrics together therefore avoids maintaining separate platform credentials and reconciling separate platform invoices for those steps. That administrative consolidation is distinct from the transport choice. Its public, self-describing discovery surface also supplies request and response schemas without a key, so a builder can inspect the contract before wiring the worker. Specified per-call metadata includes cost, vendor, latency, cache status, and request ID, supporting attribution of an LLM enrichment step to an import run. Persist that metadata alongside the application's run record before aggregating it.

Model spend is a subtotal.

Do not label it total import cost unless worker compute, storage, and any other relevant charges are also accounted for. Likewise, a missing completion record establishes missing evidence, not a zero-cost run. A worker could consume resources and stop before writing its final summary. Preserve call-level accounting as work happens, then reconcile it against the completion record. This is where a small application benefits from a narrow, explicit accounting contract more than a large collection of widgets.

Keep the application contract independent of a vendor response envelope. The code's `modelCallCostsUsd` is a local field populated by an adapter, not a promised field from every provider. That extra mapping is a little work, but it keeps a future provider change from rewriting the delivery policy. Ship the run ledger and the three decisions first; additional chart dimensions can wait until an investigation needs them.

## Which tools belong on each side of the boundary?

Compare the job each tool would own before comparing dashboards. Healthchecks, Prometheus, Grafana, and Sentry represent different choices in this workflow; they are not interchangeable boxes with a common feature score. Their documentation is linked below so the boundary can be checked before implementation.

| Option | Useful responsibility for this import | Decision boundary |
| --- | --- | --- |
| Healthchecks | Monitor scheduled check-ins and missed execution deadlines | Tie a success check-in to the intended completion condition; an early check-in cannot prove accepted output |
| Prometheus | Collect numeric operational signals and evaluate metric-based alert rules | The application still defines expected deliveries and what counts as a useful result |
| Grafana | Present operational signals and configure alerting over supported data sources | Check the chosen data source and rule behavior; a dashboard cannot infer the business meaning of an empty roster |
| Sentry | Investigate captured application errors | Error evidence explains failed attempts; it does not establish that every expected delivery arrived |

For a small application, choose the metrics API plus a separate heartbeat when that is enough to answer the operational question. Add an application-owned polling evaluator and notification path if output counts also need to trigger alerts. An empty but completed import needs that explicit result policy. Start with a positive-output expectation only for feeds where zero records actually indicates trouble; a legitimately empty delta should have a different expectation.

A team already operating Prometheus and Grafana may reasonably keep those systems and add the expected-delivery policy there. The trade-off with a metrics API plus heartbeat is that you must own the output evaluator and notification delivery; it is not a full monitoring replacement. If integrated metric alert rules are the requirement, prefer the existing alerting stack rather than assuming a metrics query sends notifications. Use an error tracker when investigating captured failures is the immediate need. None of these choices removes the need to define accepted output.

Nor does this design establish distributed tracing, crash symbolication, or session replay coverage. Those are separate requirements. This note makes a narrower decision: detect missing healthtech deliveries and retain enough numeric evidence to attribute the work they consumed.

## Check the delivery contract before switching alerts on

Before enabling notifications, give every expected import a due time, a grace interval, and an explicit output expectation. Confirm which feeds can legitimately deliver zero records. Exercise the missed, empty, partial, and delivered cases, then verify that the notification reaches the responsible operator. A console line is insufficient.

Next, retry a fixture and check both sides of the ledger: accepted records must stay distinct while attempt costs accumulate. Send the success heartbeat only after the agreed delivery condition is met. For feeds where an empty result is valid, completion can satisfy that condition; for a feed promising accepted records, process exit alone cannot. Also decide what happens when the evidence collector is unavailable, since an empty query result must not silently become a healthy import.

Finally, review the dashboard labels with the person using the alert. Show accepted records, API failure counts, duration, and backlog where useful, with model spend clearly labeled as a subtotal. Keep opaque run identifiers in the investigation evidence and patient content out of this operational example. The first useful screen should answer which scheduled delivery needs attention and what work has been attributed to it. Everything else can earn its place later.

## Further reading

References for the heartbeat, metrics, alerting, and error-investigation boundaries:

- Healthchecks documentation: https://healthchecks.io/docs/
- Prometheus metric types: https://prometheus.io/docs/concepts/metric_types/
- Prometheus alerting rules: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- Grafana Alerting documentation: https://grafana.com/docs/grafana/latest/alerting/
- Sentry error monitoring documentation: https://docs.sentry.io/product/issues/
