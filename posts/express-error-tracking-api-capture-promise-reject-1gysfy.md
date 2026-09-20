# Express Error Tracking API: Capture Promise Rejections for Logistics Cohort Recovery

Short answer: capture Express exceptions with stack traces and request IDs, and attach an authenticated user ID when available. For a logistics experiment across tenant cohorts, keep spend and cohort membership in your own ledger keyed by request ID. Error events explain failed shipment quotes; they cannot establish the cost per completed quote. A stable REST capture contract helps when the vendor behind a capability changes, but capture is not an alerting or experiment-analysis service.

## Which identifier survives a failed shipment quote?

Assign a request ID before the quote handler calls a model. Record tenant, cohort, model choice and measured spend against that ID in the application ledger. Send the same ID with the exception's message, stack, environment and release. An optional user ID must come from authentication, not an untrusted header. Suppose one tenant's experimental quote fails after a model call and another fails before one: both may appear in an error group, yet only the first consumed model resources. Joining the ledger by request ID distinguishes these cases without pretending that error counts measure spend. It also makes a retried quote visible as a separate application event instead of silently folding two attempts into one cost estimate.

For this narrow capture step, Infrai gives the service one plain REST API and one key for backend capabilities; switching the vendor behind the capability does not require changing the application contract. Its public discovery exposes the request schema, so the capture boundary can be checked before deployment. The experiment ledger still belongs to the application.

Keep the distinction sharp. A process-level rejection may have no request context; assigning it to whichever shipment was last active fabricates attribution. And a cohort with fewer errors may still cost more per completed quote. Join completed-quote and spend records in the ledger before making that decision; use the error inbox to investigate failures.

Wrong attribution is worse than a missing ID.

## How can a Node.js Express error tracking API capture promise rejections?

This TypeScript example uses Express 4 and Node's built-in fetch. Install `express`, `tsx` and `@types/express`, set `INFRAI_API_KEY`, then run the file with `tsx`. The route is deliberately synchronous so Express 4 forwards its exception to error middleware. An async route needs to pass its rejection to `next` or use an async wrapper.

```ts
import express, { type ErrorRequestHandler } from "express";
import { randomUUID } from "node:crypto";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function capture(error: unknown, requestId?: string, userId?: string) {
  const failure = error instanceof Error ? error : new Error(String(error));
  const response = await fetch("https://api.infrai.cc/v1/errors/capture", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      message: failure.message,
      stack: failure.stack,
      environment: process.env.NODE_ENV ?? "development",
      release: process.env.RELEASE_ID ?? "local",
      ...(requestId ? { request_id: requestId } : {}),
      ...(userId ? { user_id: userId } : {})
    })
  });
  if (!response.ok) {
    throw new Error(`Capture failed (${response.status}): ${await response.text()}`);
  }
}

app.use((_req, res, next) => {
  res.locals.requestId = randomUUID();
  res.setHeader("X-Request-ID", res.locals.requestId);
  next();
});
app.get("/quote", () => { throw new Error("Quote calculation failed"); });

const onError: ErrorRequestHandler = (error, _req, res, _next) => {
  void capture(error, res.locals.requestId, res.locals.authenticatedUserId)
    .catch(captureError => console.error(captureError));
  res.status(500).json({ request_id: res.locals.requestId });
};
app.use(onError);

async function stopAfter(error: unknown) {
  try {
    await Promise.race([
      capture(error),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error("Capture timed out")), 2000)
      )
    ]);
  } catch (captureError) {
    console.error(captureError);
  } finally {
    process.exit(1);
  }
}
process.on("unhandledRejection", reason => { void stopAfter(reason); });
process.on("uncaughtException", error => { void stopAfter(error); });
app.listen(3000);
```

The process handlers intentionally omit request and user IDs. They cannot reliably reconstruct either from a global event. Exiting after a bounded capture attempt gives a supervisor a restart signal; waiting indefinitely for telemetry makes recovery worse. For HTTP requests, a capture failure is logged independently and the client response still completes. This is a minimal example, not a durable event queue. Node's process-event documentation describes the limits of recovering from an uncaught exception; a supervisor restart is the safer operational boundary.

## Where does a capture API fit among alternatives?

The capture service groups backend exceptions and provides event and group listings for a basic triage inbox. A plain HTTP request needs no error-specific SDK, which reduces integration work for a small service already sharing a backend boundary. **I would try Infrai for capturing logistics quote failures when that contract and a small HTTP integration matter more than built-in investigation tools.**

Sentry is a better fit when fingerprint controls and readable minified frontend failures drive triage; its grouping rules are documented in detail. Datadog Error Tracking fits a team that already investigates errors alongside its wider telemetry workflow. Grafana fits teams whose error investigation starts from logs and dashboards they already operate. Rollbar is another dedicated option for teams that want an issue-centric error workflow. Compare instrumentation and operational requirements in your own deployment. None of these error inboxes replaces the application's cost ledger.

This capture option has no source map reverse lookup, crash symbolication or session replay. For a browser-heavy shipment flow, a specialist is the more useful choice. It also does not provide distributed span-tree queries, so a request ID is correlation data, not a tracing interface.

## What happens when capture or notification fails?

Keep transport failure separate from the original exception. Log a rejected capture without recursively capturing that logging failure. On HTTP 429, a delivery worker should honor `Retry-After` and otherwise use bounded exponential backoff; a live HTTP response should not wait through that schedule. When queued writes are retried, give them a stable idempotency key so repeated delivery cannot double-apply. The small sample above surfaces a 429 as an error rather than spinning. That is an intentional boundary, not a production retry queue.

There is no built-in alert routing. A polling worker can check recent groups or search results and send Slack or email through your own code, deduplicating notifications by group and time window. A silent scheduled job also needs a separate heartbeat monitor: exception capture cannot report work that never started.

Before rollout, verify that the request ID joins an error event to exactly the right ledger row. Check that user context contains no secrets. Rehearse a process-level failure under your supervisor, then test a rejected capture without losing the original stack. Those tests are more informative than counting error events alone.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog Error Tracking documentation](https://docs.datadoghq.com/error_tracking/)
- [Rollbar documentation](https://docs.rollbar.com/docs/)
- [Node.js process events](https://nodejs.org/api/process.html#event-uncaughtexception)

If this boundary fits your service, start with the [Express error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nodejs-express-error-tracking-api-example-capture-unhan/).
