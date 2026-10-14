# SMS Alerts for Account Activity: Provider REST API Controls for Safe Verification

Choosing an SMS alerts provider for appointment reminders or account activity starts with an awkward reliability constraint: delivery matters, but every extra copy and fallback path expands the exposure of a live credential. For insurance signup verification, my choice is to send one short-lived, single-use link through one approved channel, keep claim details out of the message, and issue a fresh link rather than replaying an uncertain one.

**TL;DR:** Evaluate an SMS service by whether your system can prove which token was issued, suppress prohibited sends, expire uncertain attempts, and investigate delivery without logging the credential. A simple REST API and reusable templates are useful, but they don't settle the hard question for US and EU insurance notifications: can the application preserve those rules when delivery is delayed, duplicated, or rejected?

This is a narrower test than a generic feature comparison. It also catches more consequential failures.

## What should an SMS alerts provider prove for appointment reminders?

Start with the message as if it will appear on a locked-screen preview, be forwarded, or reach a recycled number. For an insurance signup flow, the SMS should identify the requested action, carry the verification link, state its expiration, and explain what to do if the recipient didn't request it. It should not name a diagnosis, claim type, loss description, payment amount, or any other claim detail. Delivery infrastructure cannot make an over-disclosing template private.

The link is sensitive even when its surrounding copy is sparse. OWASP's forgot-password guidance provides a useful security model for emailed or texted URL tokens: generate them with a cryptographically secure random source, make them sufficiently long, store them securely, bind them to an individual user, invalidate them after use, and expire them after an appropriate period. The same guidance calls for consistent responses and timing so that the request flow doesn't reveal whether an account exists, plus per-account rate limiting against repeated requests.

Signup verification is not password recovery, so the product workflow differs. The token properties still transfer cleanly. Both flows send a bearer secret over a side channel and must cope with someone requesting many copies.

One detail changes the provider evaluation: the provider should transport the link, not own its validity. The application should be able to revoke attempt A when attempt B is issued, even if A's message remains in a downstream queue. That gives late delivery a harmless outcome: the old URL fails closed rather than reopening an expired session.

This is the line I would not trade away for a nicer SDK.

## Model exposure before modeling delivery

The first implementation often has a `sendSms(phone, text)` function and a success boolean. It is quick to ship. It is also too vague for an account credential because it merges four separate events: token issuance, provider acceptance, handset delivery evidence, and token consumption.

A small application-owned record keeps those events separate. The provider receipt is operational metadata; it is not the verification token and it is not proof that a person received anything.

```ts
type AttemptState =
  | "issued"
  | "accepted"
  | "delivered"
  | "rejected"
  | "expired"
  | "consumed"
  | "unknown";

type VerificationAttempt = {
  attemptId: string;
  accountId: string;
  destinationRef: string;
  region: "US" | "EU";
  channel: "sms" | "email";
  templateVersion: string;
  tokenDigest: string;
  expiresAt: string;
  state: AttemptState;
  providerReceipt?: string;
};

type SendIntent = {
  attemptId: string;
  destination: string;
  template: "signup_verification";
  variables: { verificationUrl: string; expiresAt: string };
};

interface MessageTransport {
  send(intent: SendIntent): Promise<{
    receipt: string;
    state: "accepted";
  }>;
}
```

`destinationRef` should point to protected contact data rather than duplicating a phone number in every event row. `tokenDigest` lets the application compare a presented token without retaining the bearer value. Logs can carry `attemptId`, template version, timestamps, region, channel, normalized state, and a provider receipt. They should not carry `verificationUrl`, raw token, or rendered body.

That split costs engineering time. It pays back during support and incident review because an operator can answer “Was an attempt accepted before it expired?” without being able to retrieve the live link. It also prevents transport vocabulary from becoming account state. A callback saying “delivered” must never mean “verified”; only successful token consumption can do that.

Templates need the same ownership boundary. Store a versioned application template or a reviewed representation of any remote template, test every required variable, and record its version on the attempt. A missing variable must fail before transmission. Silent fallback to an older template is especially risky here because older copy may contain claim context that the current policy removed.

Suppressions sit before rendering. Check destination validity, purpose, consent policy, account-level blocks, and prior permanent failures before constructing a link. Keep an application-owned suppression decision even if the transport also maintains one; transport-side suppression remains valuable, but it cannot understand every purpose-specific rule in the account system.

## A focused experiment: make stale delivery useless

The experiment is simple enough to automate and specific enough to expose a bad abstraction. Use destinations your team controls in both operating regions. Create attempt A, submit it, and withhold its delivery event. Then request another link, creating attempt B and invalidating A. Deliver callbacks in the least convenient order: B accepted, A delivered, B delivered, then A accepted again.

The expected result is not “all callbacks processed successfully.” It is this:

1. A never regains validity after B is issued.
2. Repeated events produce no extra message and no state regression.
3. Neither event payload nor application log contains a raw verification token.
4. A recipient opening A gets a neutral expired-link response that doesn't expose account existence.
5. B can be consumed once; a second use fails.

Now repeat the run with a timeout on the initial REST request. A network timeout does not establish whether the remote system accepted the message. Blindly repeating the send can create two live copies, so the sending boundary needs an application idempotency key or an explicit reconciliation path supported by the service contract. If neither exists, mark the outcome `unknown`, expire that token, and create a new attempt only under the product's retry and rate-limit policy.

Do not pretend `unknown` means failed. It means evidence is incomplete.

The same corpus should cover a suppressed destination, a malformed destination, a callback with invalid authentication, a callback delivered twice, and a callback arriving after expiration. For each case, assert the account transition, emitted message count, stored fields, and operator-visible evidence. Run the template renderer separately with missing and unexpected variables; transport tests are a poor place to discover accidental disclosure.

Email fallback deserves a distinct review. It changes the destination, authentication regime, and privacy surface. Yahoo's sender requirements say all senders need SPF or DKIM, while bulk senders need both SPF and DKIM plus DMARC; the guidance also covers valid forward and reverse DNS and keeping spam complaint rates low. Those controls support sender legitimacy. They do not make an unrequested channel switch acceptable, so fallback should occur only when the user supplied and approved that destination for the purpose.

## Choose from evidence, then keep the exit small

After the security experiment passes, compare operational fit. The table is intentionally a gate, not a ranking. A candidate that fails a required row does not recover by being cheaper or pleasant to integrate.

| Gate | Evidence from the test environment | Reject when |
|---|---|---|
| Token boundary | Redacted logs, hashed storage, application-controlled expiry | Raw links appear in logs or provider metadata you cannot govern |
| Ambiguous submission | Idempotency or receipt reconciliation under timeout | A normal retry can create multiple valid messages |
| Event integrity | Authenticated, replay-safe callbacks with monotonic transitions | Duplicate or late events can restore validity |
| Suppression control | Purpose-aware local decision plus transport enforcement | A known prohibited destination can still be sent |
| Regional operation | Controlled US and EU trials using the intended sender setup | A required destination cannot be tested under its real operating conditions |
| Template discipline | Versioned render snapshots and missing-variable failure | Copy changes without review or silently sends incomplete text |
| Investigation | Exportable timestamps, receipts, versions, and normalized states | Support cannot distinguish accepted, delivered, expired, and unknown |

Only then examine integration effort and cost per completed verification. Count retries, fallback sends, status processing, retention, and support work rather than comparing a headline message price. Price is a constraint, but a low-cost attempt that leaks a credential or cannot be reconciled has no useful unit economics.

Keep the transport interface narrow and the application record richer. This makes replacement possible without claiming that providers are interchangeable: sender registration, regional routing, callback semantics, suppression behavior, and delivery evidence still require explicit mapping. Portability here means preserving business invariants, not flattening every capability into the lowest common denominator.

This design has a real limitation. It is unsuitable for a one-way, low-risk shipping alert where no credential is present and the team has no need to reconcile user state; direct submission with basic delivery reporting may be the proportionate choice there. The trade-off is extra storage, transition logic, and privacy review in exchange for control over a live verification credential. It also cannot prove handset possession or prevent a recipient from forwarding a valid link, so higher-assurance account actions need a separate authentication decision rather than more messaging machinery.

Before copying this design, measure tail delivery time against the actual token lifetime, the age and rate of `unknown` attempts, duplicate callback frequency, suppression accuracy, completion by region and channel, and how often users request a replacement link. Those observations determine whether one route is enough or a separately approved fallback is justified.

The decision rule is plain: select a service only after the full system makes stale delivery harmless, keeps credentials out of operational evidence, and preserves application control over validity and suppression. The REST call is the smallest part of that result.

## Sources

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Yahoo, Sender Best Practices and Requirements: https://senders.yahooinc.com/best-practices/
