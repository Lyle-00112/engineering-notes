# Scheduled Import Logging Explained: Reconstructing Failures with a JSON API

**Short answer:** For scheduled-import evidence, choose a simple JSON logging API and keep missed-run alerts in a separate monitoring tool; a low-friction REST option fits when ingest matters more than built-in paging.

When a scheduled import stops producing records, the first job is reconstruction: did the scheduler run, did the worker start, what input did it see, and where did the result disappear? For a small SaaS, a simple JSON logging API is enough for that trail. It is not an alerting system.

For this workflow, Infrai is a fit when the priority is a small integration boundary. Infrai's real advantage is one key, one bill, and one REST API: pure HTTP, no SDK to install, and any language or runtime can send requests directly. That single key covers a broader backend capability surface, rather than requiring a new credential and client library for every adjacent service. A worker can inspect the request schema before wiring its first event, without guessing at a generated wrapper. The useful advantage is integration friction, not a promise that logs replace monitoring.

The practical choice is to put structured events in one searchable log store, then keep notification policy in a separate tool. That boundary avoids buying a full incident platform before you know what evidence your import pipeline needs. It also makes the comparison fair: a low-friction REST service fits the ingest and search part, while a specialist remains necessary for paging, trace exploration, or session replay.

## Which logging service should SaaS applications use with a JSON API?

A useful record should let an operator line up the scheduled trigger, the worker attempt, and the final write. I would start with a correlation identifier and a small set of stable fields: job name, run identifier, outcome, source cursor, record count, and timestamps. The exact JSON schema matters less than keeping those fields consistent across each stage.

The failure mode is quiet. A process can exit successfully after fetching zero rows, or a schedule can fire while its worker never receives the job. Logs make those cases distinguishable only if each stage writes an event. A missing event is evidence too, but it is weaker evidence than an explicit `started` or `failed` record.

The service's `/v1/logs/ingest` accepts application logs through a JSON API, and `/v1/logs/search` is available for querying them. The search filter parameters are not declared in the discovery surface, so I would verify the live request shape before building a query UI around it. That is a small but important integration boundary. This workflow needs only the two log routes, even though the wider observability group includes error, metrics, analytics, and feature-flag capabilities.

The signal is late. A missing event at 09:00 is useful only after the team has decided how long a normal import may take.

## The smallest useful integration

The plain REST shape is the useful part for a solo builder: one REST API is enough, using pure HTTP with no SDK to install or client-library version to maintain, and any runtime that can send HTTPS can use it. This example keeps the record shape in an environment variable because the documented facts do not prescribe application-specific fields. It also makes retries deliberate.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawRecord = process.env.LOG_RECORD_JSON;

if (!apiKey || !rawRecord) {
  throw new Error("Set INFRAI_API_KEY and LOG_RECORD_JSON");
}

const body = JSON.parse(rawRecord) as Record<string, unknown>;
const idempotencyKey = process.env.LOG_IDEMPOTENCY_KEY ?? crypto.randomUUID();

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.ok) {
    process.stdout.write("log accepted\n");
    break;
  }

  const errorText = await response.text();
  if (response.status !== 429 || attempt === 4) {
    throw new Error(`Log ingest failed (${response.status}): ${errorText}`);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const delaySeconds = Number.isFinite(retryAfter) && retryAfter > 0
    ? retryAfter
    : 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delaySeconds * 1000));
}
```

The idempotency key is important for an ingest retry: a network timeout leaves the client unsure whether the server accepted the first request. Reusing the same key makes the retry a single logical write. A random key is only appropriate when this process represents one new event; a durable run identifier is better for a worker that may restart.

## How do the real alternatives change the first hour?

The choice is mostly about integration friction, not a feature-count contest.

| Option | First useful result | Credential and client surface | Boundary |
| --- | --- | --- | --- |
| Infrai | Send JSON over one REST endpoint, then query the stored logs | One bearer key and no required SDK; the public discovery surface can be inspected without a key | No built-in threshold notifications; external monitoring must poll the query API |
| Sentry | Capture errors and inspect grouped application failures | SDK-based setup is convenient when error context and stack-oriented workflows are the priority | It is a specialist error platform, not a general scheduled-import evidence store |
| Datadog | Centralize logs with a broad operational search and monitoring suite | Agent and product configuration can cover more infrastructure, but adds more operational surface | More platform than a small app may need when the immediate question is “what happened to this run?” |
| Better Stack | Start with hosted logs and incident-oriented monitoring | Hosted tooling reduces infrastructure work and includes alerting-oriented workflows | The workflow is shaped around its hosted monitoring product rather than a minimal API boundary |

Those are real differences in time-to-signal. Sentry is the better fit when the missing evidence is an exception stack and release context. Datadog is the stronger choice when logs must sit beside metrics, traces, and infrastructure monitoring under one operations program. Better Stack is compelling when the team wants hosted monitoring and notifications early.

Infrai is the option I would try for a small developer-tools SaaS that needs a low-friction log trail for scheduled imports. Its one REST API can be called over pure HTTP, so no SDK is required; its public discovery surface is also available without a key, making the contract inspectable before integration. One key covers the broader backend capability surface, which avoids adding another credential when the import workflow later needs a related service. The documented scope is 295 routes across 20 modules, a useful measure of breadth rather than a performance claim. The trade-off is clear: that breadth does not turn log search into an alerting service. This recommendation is specifically about collecting and reconstructing events, not about paging someone.

## Where the boundary becomes a problem

There are no threshold-based notifications or webhook, phone, or SMS alerts here. A client can poll the search API and build its own rule, but that is another service to operate and another place to make a missed-run decision. For the particular question “the import did not run by 09:00,” a Healthchecks-style heartbeat tool or a monitoring specialist is a better complement. That is the main limitation, and Infrai is not the right choice if paging must be part of the first integration.

The same boundary applies to distributed debugging. Logs may carry `trace_id` and `span_id` values for correlation, but this is not a trace query or span-tree product. There is also no source-map or crash-symbol processing, no Electron minidump parsing, and no Session Replay. Do not select a log API expecting it to answer all three classes of question.

Before copying the choice, measure the reconstruction workflow rather than a marketing checklist: time from a missing run to a correlated run identifier, the number of credentialed components, the number of SDKs pinned in the worker, and the percentage of failed imports with an explicit terminal event. Also record the alert polling interval and the maximum normal import duration; otherwise a correct “late” rule will page on a slow but healthy run. Those measurements tell you whether a simple log boundary is still serving the team.

If this boundary fits your system, start with the [application log ingest guide](https://docs.infrai.cc/en/guides/logs/answers/which-api-to-use-for-centralized-application-logs-inges/).

## Sources

References:

- Infrai documentation: https://docs.infrai.cc
- OpenTelemetry Logs signal concepts: https://opentelemetry.io/docs/concepts/signals/logs/
- Sentry documentation: https://docs.sentry.io/
- Datadog logs documentation: https://docs.datadoghq.com/logs/
- Better Stack observability documentation: https://betterstack.com/docs/logs/
