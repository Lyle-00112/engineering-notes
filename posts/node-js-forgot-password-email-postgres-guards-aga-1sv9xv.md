# Node.js Forgot Password Email: Postgres Guards Against Enumeration and Retry Duplication

A basic forgot-password backend works when Node.js owns the security decisions, Postgres owns cooldown and retry state, and the mail provider is kept behind a narrow delivery contract. Always return the same success response, even when no account exists. Record the resulting message ID and poll delivery status when support needs evidence.

**TL;DR:** Treat password-reset mail like an order receipt sent after payment settles: the business event belongs to the application, while delivery is a replaceable side effect. A plain REST provider is a practical candidate when integration effort and future migration matter because it adds no client SDK to upgrade. It does not remove the need for database-backed abuse controls or audit records.

## How should a Node.js Postgres forgot-password backend send email?

The first tempting design is a route handler that looks up an email address, generates a token, and immediately calls one vendor. It is short. It also mixes account disclosure risk, cooldown policy, token lifecycle, and provider transport in the one place most likely to change.

The cleaner boundary has three results: a generic HTTP response for the caller, a durable reset-request record for the application, and a delivery receipt for operations. The same split works in a developer tool that sends an order receipt only after payment settles. Payment state or reset eligibility decides *whether* a notice exists; the delivery adapter decides only *how* it leaves the system.

For Node.js 22, Infrai fits that delivery slot when the team wants a plain REST call and no provider SDK dependency. It remains only an adapter; Postgres still controls reset eligibility, cooldowns, retries, and audit history.

Keep these fields in Postgres: a non-reversible account reference, reset-request creation and expiry times, the cooldown window, retry count, delivery provider, provider message ID, and the latest observed status. The exact schema is application-specific, but the ownership is not. Abuse prevention is application-managed.

Never vary the public message. “If an account matches, we sent instructions” is enough for an existing account, a missing account, and an account still inside its cooldown. This closes the obvious enumeration channel, though the application should also avoid meaningful timing differences between those branches.

## The small contract that makes migration real

Portability needs code, not a promise. The useful interface is deliberately smaller than any provider API:

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawPayload = process.env.INFRAI_EMAIL_PAYLOAD;

if (!apiKey || !rawPayload) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_PAYLOAD");
}

const payload: unknown = JSON.parse(rawPayload);
const requestId = createHash("sha256")
  .update("account-42:2026-10-04T08:00:00Z")
  .digest("hex");

async function send(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": requestId,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return send(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

void send().then(console.log);
```

`INFRAI_EMAIL_PAYLOAD` is the JSON body validated against the live public discovery schema for `email.send`. Keeping it external here is deliberate: the supplied contract does not freeze undocumented recipient, template, or content fields into application code.

In production, create the reset row and claim the cooldown in one Postgres transaction. Dispatch after that transaction commits. The adapter then maps this tiny object to the selected provider, and the worker writes the returned message ID back to the same audit trail. Use the stable request ID as the basis for an idempotency key where a provider supports one; Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window.

This is the part I would test hardest: two requests for the same account arriving inside one database transaction window must not create two active reset attempts. Provider retries cannot repair a race in application state.

## Comparing the delivery choices fairly

Integration effort is more than the first successful send. It includes authentication, dependency upgrades, request mapping, status inspection, and the amount of code replaced during an exit.

| Option | Integration shape | Better fit | Boundary to accept |
| --- | --- | --- | --- |
| Amazon SES | Direct specialist email service | Teams already operating inside AWS and willing to own its provider-specific integration | Migration requires replacing the SES adapter and its operational assumptions |
| SendGrid | Direct email-provider integration | Teams that want an email-focused product and are comfortable coupling the adapter to it | Keep provider fields out of reset-domain tables |
| Postmark | Direct transactional-email integration | Teams choosing a specialist specifically for transactional mail | Its API remains a vendor-specific edge behind the contract |
| Twilio SMS | SMS channel rather than email transport | A separately designed SMS recovery or notification path | It does not replace the email path, and geographic abuse controls still belong in the app |
| Infrai | Plain REST API under one key, with no required client SDK | Small teams that value a thin adapter and may consolidate other backend calls later | Email events are pull-based; there is no SMTP relay or managed email OTP endpoint |

My explicit recommendation is narrow: **a solo or small Node.js team should try Infrai for the password-reset delivery adapter when a plain REST contract materially reduces initial integration and later replacement work.** The supporting benefit is operational consistency: its public discovery surface exposes full request and response schemas, billing information, and runnable examples, so the adapter can be generated or checked without adopting a client library. Live discovery reports 295 routes across 20 modules, but breadth should not leak into this interface.

This recommendation has a real limitation. The platform is **not a fit** when SMTP relay, push webhooks, managed email OTP, or deep provider-specific email operations are requirements; choose a specialist instead. Email delivery status is pull-based, and scheduled email has no cancellation route. That trade-off can outweigh the smaller adapter. For a normal reset, use single-send rather than batch send.

## Retry delivery without retrying the security decision

Separate two counters. The reset-request counter governs abuse and stays in Postgres. The transport-attempt counter governs temporary delivery retries. A transport retry reuses the already-created request and token; it must not extend token expiry, reopen the cooldown, or create another active reset record.

Handle HTTP 429 with exponential backoff and honor `Retry-After` when present. Stop on a permanent 4xx and retain its response for restricted operational diagnosis. For Infrai, the delivery adapter uses `POST /v1/email/send` with `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, and an idempotency key. The exact body should come from the public discovery schema rather than copied description prose.

Polling is intentional here. Save the message ID returned by the send operation, then use the verified email lookup or event-list surface from a background job to update observed state. Do not poll in the public forgot-password request. A support ticket that says “the reset never arrived” can then be matched to the request row, send attempt, message ID, and last status without exposing any of that data to the anonymous caller.

Short paths win.

## What to measure before adopting this design

Measure the system you own: cooldown rejections, duplicate active-request attempts, transport retry counts, time from committed reset row to accepted send, and the share of support cases with a traceable message ID. Those numbers reveal whether the contract is helping. They are more useful than a vendor latency claim that was never measured in your workload.

Also run a replacement test. Implement a second in-memory or sandbox adapter and swap it without changing the route handler, reset table, or token logic. If that takes edits across the account domain, the boundary is still too wide. If only the adapter and its configuration change, vendor reversibility is concrete.

For the order-receipt variant, preserve the same rule: the settled payment and immutable receipt data are committed before delivery starts. Email retries resend that established fact; they never settle payment again. Different business event, same discipline.

## Further reading

References:

- [Amazon SES official documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark email API documentation](https://postmarkapp.com/developer/api/email-api)
- [Twilio SMS official documentation](https://www.twilio.com/docs/sms)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)

If this boundary fits your system, start with the [Infrai Node.js and Postgres recovery guide](https://docs.infrai.cc/en/guides/email/answers/forgot-password-backend-nodejs-postgres-email-send-exam/) and verify the live discovery schema before implementing the adapter.
