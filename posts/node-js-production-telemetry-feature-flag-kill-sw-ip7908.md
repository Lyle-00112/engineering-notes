# Agent Loop Telemetry: Runtime Switches for Latency and Token Evidence

**Short answer:** In a Node.js production incident, make the feature flag kill switch remove optional detail, never the run-level evidence that explains latency and token use. The useful test is whether an operator can prove what changed after the switch moved.

The operational constraint is evidence continuity. During a production incident, a Node.js agent loop can become noisy before it becomes obviously slow: retries multiply spans, tool arguments make records large, and a detailed event stream can obscure the few fields needed to decide what happens next. The design keeps one bounded record per run, gates enrichment at runtime, and makes the switch itself observable. That choice favors a trustworthy timeline over a richly indexed record for every step.

Before copying this pattern, measure three things in a fixed workload: required-record coverage, end-to-end latency, and the gap between observed token counts and any derived cost estimate. A quieter stream is not a successful rollback if the explanation disappeared with it.

## How should a Node.js production feature flag kill switch preserve evidence?

The switch should sit around optional telemetry, not around the agent's decision path. A disabled state can suppress detailed tool arguments, prompt fragments, or high-cardinality step attributes while leaving the request behavior unchanged. That boundary matters: changing the workload and changing its measurement are different operations, and mixing them makes an incident timeline hard to interpret.

Keep the evidence.

The required record needs a stable run identifier, start and end timestamps, outcome, bounded step counts, retry count, and token quantities when the runtime exposes them. Store an explicit marker when a cost value is estimated. Token counts are observations; a rate calculation is a separate interpretation that can age independently.

The flag update also needs an audit event: actor, reason, effective state, and time. Do not put secrets, full prompts, or unrestricted tool arguments in that event. OWASP's Logging Cheat Sheet treats sensitive-data exclusion, access control, and tamper resistance as logging concerns, not dashboard polish.

## Can the rollback be proven after the stream changes?

Yes, if the test checks records rather than a dashboard. Use a deterministic fixture with three steps, one retry, and known token fields. Start requests before the switch changes, flip the state, then start fresh requests. The expected result is not zero optional events; it is a visible boundary where new runs retain their required record and stop producing optional detail.

An in-flight run is the awkward case. Decide whether its enrichment decision is captured at run start or read for each step, then test that choice. Reading per step may react faster, but it can produce a mixed record. Capturing once gives a cleaner explanation of what the run saw. Neither is universally correct; the important part is recording the effective state alongside the run outcome.

Keep the queries boring. Compare a window before and after the change, inspect rejected records, and confirm that missing optional fields are distinguishable from a failed exporter. A sink outage and an intentional sampling decision must not collapse into the same null value.

## Which collection path keeps the evidence legible?

There are three practical paths for a small team. They differ less by brand than by where the contract lives and how much control remains during an incident.

| Approach | Access method (SDK/REST) | Setup effort | Best fit | Main limitation |
| --- | --- | --- | --- | --- |
| In-process structured event | SDK or local library | Low | A single service with a small required schema | Process failure can lose buffered detail |
| Trace-context export | SDK with a standard trace context | Medium | Correlating model, tool, and retry spans across services | Sampling can remove detail unless required fields are retained separately |
| Control-plane flag plus event sink | REST for state, SDK for events | Medium | An emergency switch with an independent audit trail | Two paths need clear authentication and consistency rules |

The table is a decision aid, not a recommendation for one product. W3C Trace Context defines propagation fields for correlating work across boundaries, while the OpenTelemetry specification describes common telemetry signals and their semantics. Those standards help keep the event contract portable even when the collector or backend changes.

## A small Node.js control surface

The control surface can be deliberately plain. The flag provider is an interface so the runtime does not know whether state comes from a local cache, a control service, or a test fixture. The required sink is called regardless of the enrichment state.

```ts
type RunEvidence = {
  runId: string;
  startedAt: number;
  endedAt: number;
  steps: number;
  retries: number;
  inputTokens?: number;
  outputTokens?: number;
  estimatedCost?: number;
};

interface RuntimeFlag {
  enabled(name: string): Promise<boolean>;
}

interface EventSink {
  write(event: Record<string, unknown>): Promise<void>;
}

async function recordRun(
  evidence: RunEvidence,
  flags: RuntimeFlag,
  requiredSink: EventSink,
  detailSink: EventSink
): Promise<void> {
  const enrichmentEnabled = await flags.enabled("agent.telemetry.detail");

  await requiredSink.write({
    ...evidence,
    latencyMs: evidence.endedAt - evidence.startedAt,
    enrichmentEnabled
  });

  if (enrichmentEnabled) {
    await detailSink.write({
      runId: evidence.runId,
      boundedStepAttributes: collectBoundedAttributes(evidence)
    });
  }
}
```

The code leaves two deliberate edges. First, `collectBoundedAttributes` must enforce a size and field allowlist before serialization. Second, the flag read and the required write need failure behavior that is tested separately. A telemetry flag should not turn a successful agent run into a failed request merely because enrichment is unavailable, but the required record may need a stronger delivery guarantee. That is a product decision for the runtime owner, not a property to hide behind a catch block.

Run the same fixture with enrichment enabled and disabled before shipping. Record p50 and p95 run latency, required-record coverage, optional-event volume, exporter rejection rate, and the percentage of runs with observed token fields. The comparison should include a process restart and one temporary sink failure. Those cases expose buffering assumptions that a healthy local test will miss.

The trade-off is explicit: a 3-step fixture with 1 retry gives a repeatable boundary test, but it does not predict production volume. Capacity and latency still need measurements from the deployed runtime.

I would also compare three questions against the resulting data: Which step consumed the time? Did a retry change the outcome? Which token quantities were actually observed? If the answer requires reconstructing a request from unbounded logs, the schema is carrying noise. If it cannot answer any of them after enrichment is off, the required record is too small.

## The decision rule

Keep a field in the required record when it changes an operational decision. Put it behind the switch when it is sensitive, high-cardinality, or expensive to retain. Attach the effective state to every required record so an empty detail stream has an explainable cause.

This is the useful shape of a production rollback API: one authenticated state change, one audit event, and a record contract that survives the change. It does not replace deployment rollback or retention policy. It gives an AI agent loop a continuous account of latency and token evidence while optional noise is being shed.

## Sources

References:

- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- W3C Trace Context Recommendation: https://www.w3.org/TR/trace-context/
- OpenTelemetry Specification: https://opentelemetry.io/docs/specs/otel/
- Datadog pricing page, documenting separate log ingestion and indexing dimensions: https://www.datadoghq.com/pricing/
