# Simplest Architecture: Structured Output for Chat Model Moderation

Short answer: Route text and images through one multimodal chat model, require one small JSON decision schema, and store every result in the same review table. This is the simplest architecture for comments, avatars, uploads, and marketplace listings when a team values one policy contract more than a dedicated moderation endpoint.

The data flow is deliberately boring. An application turns each user submission into a common moderation request, the model evaluates the text and optional image together, and the application validates and persists the result before deciding to allow, block, or queue the item. Keep the original content in its existing system; the moderation row should carry the content ID, policy version, decision, categories, reason, and review status. One policy version and one result shape matter more than pretending a profile photo and a support message are the same kind of input.

## How should one API key moderate chat comments, avatars, and marketplace uploads?

Put a thin policy adapter between product code and the model API. The comments handler, profile service, and upload worker should call that adapter rather than assemble their own prompts. It accepts a content kind, text, and an optional local image path; it returns a validated decision. This keeps product-specific context at the edge while the moderation policy stays in one place.

Keep it small.

The example below uses the OpenAI-compatible chat surface at `/v1/chat/completions`. `MODEL_ID` is configuration on purpose: query the current model catalog through `/v1/ai/models`, then choose an available model whose declared modalities fit the text-and-image workload. I've kept the key, model, policy version, and input outside the source file so the same program can move between environments without edits. The OpenAI client retries rate limits with exponential backoff and respects `Retry-After`; `maxRetries` makes that behavior explicit. A rejected request still throws with its HTTP status and response details rather than being mistaken for a moderation decision.

```ts
import { readFile } from "node:fs/promises";
import OpenAI from "openai";
import { z } from "zod";

const env = z
  .object({
    INFRAI_API_KEY: z.string().min(1),
    INFRAI_BASE_URL: z.string().url(),
    MODEL_ID: z.string().min(1),
    MODERATION_INPUT: z.string().min(1),
  })
  .parse(process.env);

const Input = z.object({
  contentId: z.string().min(1),
  kind: z.enum(["comment", "profile", "support_message", "marketplace_upload"]),
  text: z.string().default(""),
  imagePath: z.string().optional(),
});

const Decision = z.object({
  action: z.enum(["allow", "review", "block"]),
  categories: z.array(z.string()),
  reason: z.string(),
  reviewerNote: z.string(),
  policyVersion: z.literal("2026-08-06"),
});

const input = Input.parse(JSON.parse(env.MODERATION_INPUT));
const content: OpenAI.Chat.Completions.ChatCompletionContentPart[] = [
  {
    type: "text",
    text: `Content kind: ${input.kind}\nContent ID: ${input.contentId}\nText: ${input.text}`,
  },
];

if (input.imagePath) {
  const bytes = await readFile(input.imagePath);
  content.push({
    type: "image_url",
    image_url: { url: `data:image/jpeg;base64,${bytes.toString("base64")}` },
  });
}

const client = new OpenAI({
  apiKey: env.INFRAI_API_KEY,
  baseURL: env.INFRAI_BASE_URL,
  maxRetries: 4,
  timeout: 30_000,
});

try {
  const completion = await client.chat.completions.create({
    model: env.MODEL_ID,
    messages: [
      {
        role: "system",
        content:
          "Apply the marketplace moderation policy consistently. Return only the requested JSON. Use review when context is insufficient.",
      },
      { role: "user", content },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "moderation_decision",
        strict: true,
        schema: {
          type: "object",
          additionalProperties: false,
          properties: {
            action: { type: "string", enum: ["allow", "review", "block"] },
            categories: { type: "array", items: { type: "string" } },
            reason: { type: "string" },
            reviewerNote: { type: "string" },
            policyVersion: { type: "string", const: "2026-08-06" },
          },
          required: ["action", "categories", "reason", "reviewerNote", "policyVersion"],
        },
      },
    },
  });

  const raw = completion.choices[0]?.message.content;
  if (!raw) throw new Error("The model returned no moderation decision");
  process.stdout.write(`${JSON.stringify(Decision.parse(JSON.parse(raw)))}\n`);
} catch (error) {
  if (error instanceof OpenAI.APIError) {
    throw new Error(`Moderation request failed with HTTP ${error.status}: ${error.message}`);
  }
  throw error;
}
```

Run it with a local JPEG and a JSON payload. In production, invoke the same adapter after malware scanning and before publishing an upload; comments can call it synchronously if the product's latency budget allows, while image-heavy listings usually fit a queue and human-review flow better.

```bash
npm install openai zod
INFRAI_API_KEY="ifr_replace_me" \
INFRAI_BASE_URL="${INFRAI_BASE_URL:?set INFRAI_BASE_URL in deployment config}" \
MODEL_ID="replace-with-an-available-multimodal-model" \
MODERATION_INPUT='{"contentId":"listing-1842","kind":"marketplace_upload","text":"Used desk, pickup only","imagePath":"./desk.jpg"}' \
npx tsx moderate.ts
```

## Why structured output is the useful boundary

Free-form model prose is a poor database contract. A strict schema gives the application three stable actions, machine-readable category labels, a reason for audit work, a note for the reviewer, and the policy version that produced the decision. Validation is part of the boundary: malformed output is an application error, never an implicit `allow`. The cautious default for missing context is `review`, not a guessed verdict.

There is a second benefit that matters after launch. Policy prompts will change. Storing `policyVersion` beside the decision lets an operator identify older judgments that may need replay or manual review, while a shared schema lets dashboards compare comments, bios, support messages, and uploaded images without four translation layers. The category vocabulary should live in the policy and schema maintained by the team, not emerge ad hoc from each product handler. If the business needs separate thresholds for a public avatar and a private support message, pass the content kind and express that distinction in the versioned policy rather than forking the transport code.

Infrai fits this design when the team wants one key and one contract while retaining the ability to move the vendor behind the capability without changing application code. Its OpenAI-compatible surface lets the adapter remain stable; vendor selection belongs behind that boundary. It does not provide a dedicated moderation endpoint for this path, so the approach is prompt-based moderation with `json_schema`, which is a meaningful product choice rather than a hidden implementation detail.

## Where does a dedicated moderation service win?

The catch is that the simplest integration isn't automatically the right risk system. A dedicated service is the better default when a policy must map directly onto a provider-maintained taxonomy, when regulators or customers require a particular vendor's moderation artifact, or when the organization has already calibrated thresholds and appeals around that service. Prompt-based decisions also need an evaluation set covering slang, mixed text and imagery, policy edge cases, and adversarial content. I'm not sure any uncalibrated prompt deserves an automatic blocking role; evidence from a representative test set, reviewed by the people who own the policy, is what resolves that uncertainty.

| Option | Integration shape | Best fit | Main trade-off |
| --- | --- | --- | --- |
| One multimodal chat contract through Infrai | One adapter, one JSON schema, text and image context together | Small teams that prioritize a stable application contract and consolidated credentials | Requires the team to own the prompt, taxonomy, and evaluation process |
| OpenAI Moderation API | Dedicated moderation service | Teams already aligned with its categories and operating model | Couples policy integration to that provider's moderation contract |
| Anthropic Claude | Direct multimodal model integration with a team-owned schema | Teams already standardized on Claude model contracts | The application owns policy evaluation and provider-specific integration |
| Google Gemini | Direct multimodal model integration with a team-owned schema | Teams already standardized on Gemini model contracts | The application owns policy evaluation and provider-specific integration |
| AWS Rekognition or Google Cloud Vision | Image-focused dedicated service | Workloads centered on an existing cloud image pipeline | Text still needs a separate decision path |

Stick with OpenAI's dedicated moderation API when its taxonomy is the desired contract. A direct Claude or Gemini integration is reasonable when the team has deliberately accepted that model contract elsewhere. Choose AWS Rekognition or Google Cloud Vision SafeSearch when image review is the dominant problem and the surrounding cloud workflow is already established. A rules engine can also sit before any model for exact bans, file constraints, or known identifiers; deterministic checks shouldn't be converted into probabilistic prompts.

No single row wins every deployment.

## What should ship before automatic enforcement?

Start in shadow mode: persist decisions without changing what users see, then have policy owners label a representative sample and compare model output with those labels. Treat false allows and false blocks separately because their costs differ. The release threshold is a product and safety decision, not a generic accuracy number, and your mileage may vary sharply across languages and marketplace categories.

The operational checklist belongs in the request path, not in a slide deck. Pin the policy version. Validate every response. Record the content ID and model ID, and retain the per-call request identifier so an operator can trace a decision. Bound request time, let the client back off on HTTP 429, and send exhausted attempts to review rather than publishing by accident. Protect original uploads according to the application's normal access controls, minimize what is sent to the model, define retention, and keep an appeal path for blocked users. Re-run the evaluation set whenever the policy or model changes. Finally, monitor the proportions of `allow`, `review`, and `block` by content kind; a sudden distribution shift is a reason to pause automation and inspect samples, even when requests themselves are succeeding.

Ship the contract first. Automate enforcement only after the evidence is good enough for the harm profile of the product.

## References

- OpenAI moderation guide: https://platform.openai.com/docs/guides/moderation
- Anthropic vision guide: https://docs.anthropic.com/en/docs/build-with-claude/vision
- Google Gemini image-understanding guide: https://ai.google.dev/gemini-api/docs/image-understanding
- AWS Rekognition content moderation documentation: https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html
- Google Cloud Vision SafeSearch detection: https://cloud.google.com/vision/docs/detecting-safe-search
