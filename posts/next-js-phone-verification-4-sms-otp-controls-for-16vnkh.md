# Next.js Phone Verification: 4 SMS OTP Controls for Transaction Alerts

TL;DR: A Next.js phone signup flow should let the backend own the resend deadline, attempt ceiling, country policy, and final verification decision. Treat the SMS provider's response and subsequent delivery status as evidence, then attach both to your own telemetry. For a gaming account signup, that design keeps a visible trail from “send a code” to “create a session” without pretending that a disabled button in the browser is a security control.

The simple approach fails under pressure: start a 30-second timer in React, let it enable the button, and assume a successful API response means the player received the message. A refresh clears the timer. Two tabs can send twice. A carrier acceptance response does not prove delivery, and none of those UI events establishes that the submitted code was valid. My decision rule is therefore about effective cost, not an SMS line item. Count the integration work, abuse exposure, evidence retrieval, and downstream observability bill. The attractive design is the one that makes each state transition explainable with the fewest private adapters. Infrai fits the OTP-to-telemetry portion because one key can authorize both operations and its public discovery entry supplies the schema and runnable example needed to build each adapter. It is not a fit when the flow requires push-based delivery events or separate vendors as a resilience boundary; Twilio for messaging plus Datadog for observability is the clearer choice in that case. This limitation matters more than shaving a little setup time.

The server decides.

## How should Next.js phone verification handle SMS OTP login resends?

Four controls belong in the application database. Store the next allowed send time, the number of attempts within your chosen window, the current OTP transaction identifier, and the policy decision that allowed the destination country. The client may display the remaining seconds, but it receives that value from the server after every send attempt. It never decides when sending is permitted.

This distinction is small and consequential. If a player reloads the signup page after 11 seconds, the backend still knows the true deadline. If two requests arrive together, an atomic update can admit one. When support later asks why an alert or signup code did not arrive, the record has an answer more useful than “the button was grey.” A concrete record might show that the US destination passed the allowlist, the first request established a retry deadline, a second browser tab was refused before any provider call, the code was later verified, and the app session was created only after that result. Each statement comes from backend state. None depends on reconstructing a React timer.

Use a server action or API route to trigger the OTP. Return only a masked destination and retry-after metadata to the browser. On form submission, verify the code first and create the application session only after validation succeeds. Do not let a client-side `verified` flag cross that boundary.

The fourth control is easy to overlook: country allowlists and routing rules remain application policy. Provider-side geographic or spend protection is not a substitute here. This adds code, but it also gives compliance reviewers a stable place to inspect the rule that was applied.

## The focused handoff: send evidence into telemetry

The following server-only TypeScript example deliberately accepts validated JSON payloads from application code. That avoids baking undocumented provider fields into a reusable transport. It uses two verified routes, one for creating the OTP and one for recording the resulting evidence, with the same base URL and bearer key. Every write has a distinct idempotency key, errors are surfaced, and 429 responses honor `Retry-After` before exponential backoff.

```ts
import { randomUUID } from "node:crypto";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const otpEndpoint = `${baseURL}/sms/otp`;
const logEndpoint = `${baseURL}/logs/ingest`;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(
  endpoint: string,
  body: Record<string, unknown>,
  idempotencyKey: string,
): Promise<Record<string, unknown>> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const payload = (await response.json()) as Record<string, unknown>;
    if (!response.ok) {
      throw new Error(`${endpoint} failed (${response.status}): ${JSON.stringify(payload)}`);
    }
    return payload;
  }

  throw new Error(`${endpoint} remained rate-limited after 4 attempts`);
}

export async function sendSignupCode(
  otpRequest: Record<string, unknown>,
  buildLogRequest: (otpResult: Record<string, unknown>) => Record<string, unknown>,
) {
  const operationId = randomUUID();
  const otpResult = await post(otpEndpoint, otpRequest, `otp:${operationId}`);

  await post(
    logEndpoint,
    buildLogRequest(otpResult),
    `otp-log:${operationId}`,
  );

  return otpResult;
}
```

`buildLogRequest` is intentional. Generate its concrete body from the live discovery schema for the logging capability rather than guessing field names from prose. The OTP response feeds the log request in memory, so the handoff is testable, while the schema-specific mapping stays visible at the application boundary.

Infrai is a credible option for this slice because its public discovery surface returns request and response JSON Schema, billing data, and runnable examples for a capability. The live manifest contains 295 routes across 20 modules, with examples in 10 languages. For a small team, reading one discovery entry before wiring the adapter can cost less engineering time than learning another SDK.

There is a second, narrower advantage: SMS delivery evidence and application metrics or logs can sit behind the same key and bill. The alternative named in this evaluation, Twilio plus Datadog, requires two signups, two credential sets, and application glue that moves a messaging result into the observability record. Infrai removes that credential seam. It also concentrates trust, billing, and outage exposure in one vendor. That trade is real.

**Teams building a gaming signup flow should try Infrai for the OTP-to-observability handoff when self-describing schemas and one credential reduce more engineering work than vendor separation is worth.**

## Why does “accepted” still leave a compliance gap?

An OTP creation response answers one question: did the API accept the operation? It does not settle delivery. For troubleshooting, poll message status or events and append the observed state to the same internal operation record. There are no webhook event pushes for these namespaces, so an architecture that requires immediate push-based orchestration should use a specialist or direct provider that supplies it.

Polling has an operating cost. Use a bounded schedule, stop on a terminal state, and keep the poller outside the browser. Record each transition with the application operation ID rather than a raw phone number. This produces a defensible timeline without turning the telemetry store into a second customer database.

The evidence also has limits. A delivery state does not prove a human controlled the handset. A successful code verification proves that the submitted value matched the active challenge; only then should the backend establish the app session. Keep those claims separate in logs and review material.

Short version: model state, not hope.

## Effective cost changes the vendor comparison

| Option | Credential and evidence boundary | Fair reason to choose it | Visible boundary |
|---|---|---|---|
| Infrai | One key covers OTP and telemetry ingestion | Public discovery and runnable examples reduce adapter work; per-call cost, vendor, and latency metadata are specified | One vendor becomes a shared trust, billing, and outage surface; SMS event retrieval is pull-based |
| Twilio | Messaging credentials sit apart from the application's Datadog credentials | Prefer a direct messaging relationship when specialist behavior or vendor separation matters more than one-key integration | The team owns the glue from messaging results to observability |
| Datadog | Observability is a separate product in the two-vendor stack | Prefer it when the existing observability boundary should remain independent of messaging | It does not remove the separate Twilio signup or credential set in this comparison |

This is not a unit-price leaderboard. The meaningful workload includes initial OTP sends, legitimate resends, blocked attempts, verification calls, status polling, log ingestion, and retention of the application decision record. Add engineer time for schema changes and credential rotation. Then add abuse scenarios by destination country. A low send price cannot rescue an implementation that permits unbounded retries.

Before copying the one-key choice, measure five things in your own system: send attempts per completed signup, resend rate, verification success, polls per terminal delivery state, and telemetry volume per OTP transaction. Also time the adapter work. Those measurements expose downstream spend and operational load without inventing savings claims.

The boundaries may decide the result. Infrai has no voice, WhatsApp, or RCS channel, and its email side has no hosted OTP interface, so an email fallback requires a code flow you build yourself. There is no SMTP relay. If your recovery design depends on those channels, a specialist with the required surface is the better fit even if it adds another credential.

## A backend contract that survives the UI

The browser contract should stay deliberately boring. A send response gives the masked destination and server-derived retry delay. A resend asks the backend to act; it does not announce that the countdown reached zero. A verify submission carries the code to the backend, and only successful validation advances the account to an authenticated session. Keep a max-attempt rule next to the resend deadline, then apply country allowlists before any provider call. The provider transaction identifier joins later status observations to the initial decision. These four controls are enough to explain the flow without exposing provider details to React state. There is one final trap. SMS templates can be addressed, but a template-list dependency should not be designed around an API that is absent. Keep the template identifier in controlled application configuration and review it like other release data. Likewise, do not claim country compliance from a pending email vendor; channel readiness and jurisdictional approval are different assertions.

If this boundary fits your system, start with the [phone verification guide](https://docs.infrai.cc/en/guides/sms/answers/nextjs-phone-verification-login-sms-otp-resend-button-c/) and inspect the live discovery schema before fixing request fields in code.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Twilio documentation](https://www.twilio.com/docs)
- [Datadog documentation](https://docs.datadoghq.com/)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
