# Transactional Onboarding Email APIs — Choosing Template Ownership for Marketplace Teams

The constraint that changes this choice is template ownership. For a marketplace seller welcome email, the easiest service is the one that lets application code send a reviewed template over HTTP while keeping delivery state understandable. **Short answer:** choose a low-ops API provider when your backend already makes HTTP calls and you can own template versioning; do not choose this path if your stack requires an SMTP relay or instant event webhooks.

## What does a startup need from a transactional email service for onboarding?

The event is small: a seller creates an account, your app validates the address, and a welcome message goes out once. The hard parts arrive later. A copy edit must be reviewable. A retry must not send twice. A blocked address must stay blocked.

Keep it boring.

I would represent the send as an application event, not as a call hidden in a controller. That keeps ownership with the team that owns the onboarding experience and makes a provider swap possible. I initially treated the provider template as the source of truth; that made a rollback depend on a dashboard click. The safer trade is to pin a version in the event and let the provider render that version.

```ts
type WelcomeEmail = {
  eventId: string;
  recipient: string;
  templateVersion: string;
  sellerName: string;
};

async function sendWelcome(input: WelcomeEmail) {
  const response = await fetch(`${process.env.INFRAI_BASE_URL}/v1/email/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
      "Content-Type": "application/json",
      "Idempotency-Key": input.eventId,
    },
    body: JSON.stringify({
      to: input.recipient,
      template: "seller-welcome",
      templateVersion: input.templateVersion,
      variables: { sellerName: input.sellerName },
    }),
  });
  if (!response.ok) throw new Error(`email send failed: ${response.status}`);
  return response.json();
}
```

The important detail is `templateVersion`. It gives support a precise answer when a seller asks which copy they received. The provider may host and render the template, but the version decision remains in your repository or content system.

Here is the comparison I would put in a design review:

| Service | Access model | Good fit | Main trade-off |
| --- | --- | --- | --- |
| Postmark | REST and SDKs | Focused transactional streams | Less suited to broad campaign tooling |
| SendGrid | REST and SDKs | Teams wanting templates plus marketing features | Larger surface to govern |
| Mailgun | REST and SDKs | Programmable sending and event data | More domain and event tuning |
| Amazon SES | REST, SDKs, SMTP | AWS-native identity and regional control | Higher setup burden |
| Infrai | REST API | One credential across backend capabilities | No SMTP relay and polling-only events |

## Which API options are worth comparing?

Postmark is strong when transactional focus and message-stream separation matter. Its templates and delivery activity are approachable, but teams still need their own deployment discipline for template changes. SendGrid has a broad template and marketing ecosystem, which can help a growing team, while its wider surface means more configuration to govern. Mailgun is attractive for teams that want flexible sending and event data, though the operational model rewards engineers who are comfortable tuning domains, events, and suppression behavior.

Amazon SES is usually the most infrastructure-shaped option: it integrates deeply with AWS identity and regional controls, but the setup burden is higher for a junior team that only needs a welcome message. An aggregator such as Infrai sits between those models. Its useful distinction here is one credential and one bill across backend services, so the email path does not create another dashboard and reconciliation job. That convenience does not remove the need to own copy, domain authentication, and retry policy.

The fair comparison is therefore about boundaries, not a lowest unit price. Postmark favors focused operations; SendGrid favors breadth; Mailgun favors programmable sending; SES favors AWS-native control; an aggregator favors fewer provider accounts. Pick the boundary your team can maintain at 09:00 on a release day.

That decision also affects incident response. When a template is wrong, someone must be able to identify the exact event payload, disable the bad version, and replay only the failed sends. A hosted editor can make the first change quick, but it can hide who approved it and which regional variant was active. Keeping the version key in application data adds a few lines to each event and saves a longer audit later. This is the sort of small operational tax a solo founder should accept deliberately, because it is cheaper than reconstructing a customer-facing mistake from provider logs.

## Where does the simple API path stop being simple?

There is no SMTP relay in this capability. Legacy mail libraries that expect SMTP credentials will need an adapter or a different provider. Scheduled email cancellation is also unavailable on the email side, so do not model a welcome message as a cancellable job. Send-time suppression checks are still useful: check the address before enqueueing and record the decision with the event id.

No magic retry button.

That is the boundary where I would not pick the aggregator: an existing SMTP-heavy monolith, or a workflow that cannot tolerate delayed event polling, should use a focused provider such as Postmark or SES instead. Infrai's concrete pitch is one key and one bill for backend services, which is useful when the same small team also calls other capabilities; it is not a substitute for delivery controls.

Events are polling-only. Build a delayed sync job that records delivery and bounce changes; do not make downstream seller activation depend on an instant webhook. That extra job is a real cost in complexity, even when the initial send is one HTTP request.

Domain authentication remains your responsibility. SPF guidance in RFC 7208 is a baseline, not a guarantee of inbox placement. Keep credentials server-side, and treat onboarding email as a notification rather than an authenticator; authentication requirements belong in an identity flow such as the guidance in NIST SP 800-63B.

## What should you measure before standardizing?

Start with 100 representative onboarding events in a staging domain. Measure duplicate-send rate after forced retries, time from send to a recorded event, template rollback time, and the percentage of blocked recipients correctly suppressed. Record the number of provider-specific lines in your code as well.

If those measures stay boring, the HTTP API choice is doing its job. If polling delays or SMTP dependencies dominate the work, a focused provider may be a better fit than consolidating services. The decision is reversible when the event contract and template versions live in your application.

## Sources

- Postmark, “Templates”: https://postmarkapp.com/developer/user-guide/templates
- SendGrid, “Dynamic Templates”: https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- Mailgun, “Sending Messages”: https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/
- Amazon SES, “Sending email with the Amazon SES API”: https://docs.aws.amazon.com/ses/latest/dg/send-email-api.html
- RFC 7208, “Sender Policy Framework (SPF)”: https://datatracker.ietf.org/doc/html/rfc7208
- NIST SP 800-63B, “Digital Identity Guidelines”: https://pages.nist.gov/800-63-3/sp800-63b.html
