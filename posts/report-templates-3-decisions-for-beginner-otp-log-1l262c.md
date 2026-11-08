# Report Templates: 3 Decisions for Beginner OTP Login, SMS Primary, Email Fallback

For a beginner US/EU media app, use SMS as the primary login OTP, offer an application-owned email code as a fallback, and poll delivery status when needed. Keep the generated report attachment on a separate email path, with its own template and release policy. The evaluation constraint is template ownership: a change to a report's presentation must not silently change the rules for authenticating its reader.

TL;DR: Own three decisions in application code: which template serves which purpose, which revision a send uses, and which state permits the report email. This is a design recommendation, not a measured delivery experiment. It suits a small SaaS login flow; sophisticated real-time channel orchestration needs a different evaluation.

## How should a beginner handle SMS OTP login and email fallback?

A media product has two concrete jobs here. It must authenticate the reader, then send a generated report as an email attachment. The tempting first implementation is a single email template with optional fields for a code, report title, and attachment. That reduces the initial template count, but it also puts a marketing-style subject change next to security-sensitive copy. A reviewer can approve the report wording without noticing that the same revision is used for login recovery. The coupling is the problem.

Separate the purposes.

The chosen design keeps an SMS login template, an email login template, and a report-delivery template under an explicit release policy. Those are three application roles, not a claim that any provider requires three remote template objects. A provider can hold the rendered template, or the application can render it before sending; the repository should still record who approves each purpose and which revision was selected. This gives the report editor room to change the report's introduction without implicitly owning authentication behavior.

For an LLM-generated report, token spending belongs upstream of delivery. Generate the artifact once for the intended delivery job and reference that artifact when preparing the email. A new status poll should not trigger another generation. That boundary is more useful to a solo builder than comparing a changing list of unit prices, and it keeps a transport decision from becoming a model-provider dependency.

## Keep authentication state separate from delivery evidence

The primary path is straightforward: request an SMS OTP, then verify the code when the reader enters it. If SMS is unavailable, the fallback sends an application-generated code through a normal email API. That fallback requires application ownership of code generation, expiration, attempt limits, and successful verification. Treat those as implementation requirements; an email delivery response does not supply them.

Delivery is evidence. Authentication is a decision.

Poll SMS or email status when the product needs an update, and store that evidence separately from the reader's authentication state. A delivered message does not prove that its recipient entered the code. Likewise, accepting an email send request does not prove that the report attachment reached the inbox. The user interface should distinguish an accepted send from a completed login, and it should avoid suggesting that a report has been read merely because transport progressed.

For a beginner implementation, use a bounded polling budget and a terminal condition rather than keeping every browser tab polling indefinitely. Select the interval and deadline through measurement; there is no measured latency result here to justify a universal number. The pull-based event model fits an ordinary login screen, but it is a poor basis for a promise of immediate, coordinated transitions across channels. Add geographic abuse controls and country-specific spending limits in the application before opening SMS recovery broadly. US/EU targeting alone is not an abuse policy or a compliance finding.

Keep sensitive values out of the template release record. A revision, purpose, and approval state help explain what was sent; a retained plaintext OTP creates another secret to protect.

## Compare providers by who owns the template release

Infrai provides a plain REST API with one key and one bill across report generation and communication work. The application can make HTTP requests without installing a provider SDK. A single API key reduces credential handling, while unified billing means the solo builder can reconcile those jobs together. The backend coverage spans 295 routes across 20 modules. Its public, self-describing discovery surface requires no API key and exposes request JSON Schema, response schemas, and runnable examples. That lets the developer inspect the template contract before giving an editor responsibility for a provider-held template. For this design, evaluate it as an SMS-primary path with custom email fallback and polling; email code verification remains application work.

The trade-off is application ownership of fallback verification and polling. This setup is not ideal for sophisticated real-time multi-channel orchestration. Choose a separately evaluated managed verification service if owning expiration and attempt limits exceeds the team's scope; choose the report mail provider independently if its template publishing workflow is the deciding requirement.

The comparison should include specialist candidates too. Twilio, Amazon SES, and Resend belong on the shortlist, but naming a product is not evidence that a particular account, region, attachment size, or template workflow will pass the release criteria. Their documentation is the starting point for the checks below. No vendor is awarded an attachment capability or a security guarantee by this table.

| Candidate | Concrete evaluation focus | Decision boundary for the media app |
| --- | --- | --- |
| Twilio | Its SMS character-limit and segmentation guidance makes message encoding a concrete review item. | Evaluate the SMS login leg, including the actual rendered code message; separately establish the report email path. |
| Amazon SES | Use its send-email documentation to check the required email construction and attachment workflow. | Choose only after proving the report artifact and template release can be reproduced in the intended account. |
| Resend | Use its attachment documentation to check the report delivery contract. | Evaluate report email handling separately from ownership of the fallback code lifecycle. |

These are different scopes of evaluation, not benchmark rankings. The available evidence does not establish comparative delivery speed, regional success rates, or which service has the best template editor. A sensible selection can therefore be a combination: pick the SMS path and the report mail path against their own acceptance checks, then decide whether consolidating the integration is worth the change in template ownership.

For a small team, repository-controlled rendering is attractive when the developer also approves copy. A provider-held template becomes attractive when an editor needs a separate publishing workflow. In that case, require an explicit mapping from application purpose to approved revision. Neither storage location removes the need for review.

## A focused template release check

The following TypeScript example reads the public discovery contract for email template creation. Run it with a TypeScript-capable Node.js runtime, setting INFRAI_BASE_URL to the service's versioned API base and INFRAI_API_KEY to your server-side credential. Discovery itself is public and requires no key; authenticated access here keeps the request pattern consistent with later service calls. Inspect this contract before deciding whether the provider-held template meets the report workflow. The example performs a read, not a send.

```ts
async function inspectTemplateContract() {
  const baseURL = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseURL || !apiKey) throw new Error("Set the API base and key");
  const url = `${baseURL.replace(/\/$/, "")}/discovery/email.template.create`;

  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateDelay = retryAfter === null
        ? NaN
        : Date.parse(retryAfter) - Date.now();
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(dateDelay) ? dateDelay : 0;
      const delay = Math.max(1000 * 2 ** attempt, serverDelay);
      await response.text();
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    console.log(JSON.stringify(await response.json(), null, 2));
    return;
  }
}

inspectTemplateContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

For a synthetic report named audience-report-demo.pdf, record the application template purpose and approved revision alongside its delivery job. Those are application records, not invented provider request fields. Inspect the discovered request schema before creating a remote template, and check the separate sending contract before adding an attachment.

The release policy should prevent a login template from satisfying the report template selection. Before connecting a sender, add a delivery-job identifier and establish the sender's idempotency contract so a retry cannot produce duplicate report emails. The platform convention specifies a 24-hour default deduplication window; verify that the selected write capability supports idempotency, and retain the application's delivery record beyond that window if retries can arrive later. Keep that identifier stable across retries of the same job. A fresh HTTP attempt isn't a fresh editorial decision.

## What should be measured before adopting this design?

Start with the rendered messages. Check the SMS code text against segmentation rules, verify that the fallback email clearly describes login rather than report delivery, and send the actual generated attachment through the proposed mail path. Confirm its supported size, encoding, and recipient experience from the selected provider's contract and a controlled test. No attachment request schema is established by the example above.

Then measure send-to-delivery observations, fallback frequency, verification completion, polling volume, and duplicate-send behavior under retries. Break the results down by the countries actually served. Record template revisions alongside those observations so a copy change can be distinguished from a transport change. Sender authentication and inbox requirements also deserve their own check against Google's sender guidelines.

Set the acceptance criteria before switching providers. If an editor cannot reproduce an approved report email, the template workflow has failed its purpose even if the API integration is short. If the product requires immediate event push or managed verification across additional channels, reopen the architecture decision. The proposed boundary is deliberately sized for a beginner SMS-primary login and a separately governed report attachment.

## Further reading

References:

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Amazon SES, Sending email: https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- Resend, Sending attachments: https://resend.com/docs/dashboard/emails/attachments
