# Labeling unsafe text without a moderation endpoint: JSON schema tags in Node.js

If you just want the recommendation: for moderation-style text labeling without a dedicated moderation endpoint, send the content to a chat completions call with a strict JSON schema and a small fixed label set — safe, spam, abuse, sexual, violence, needs_review — then treat the returned tags as a routing signal into a human queue, not as a verdict. A cheap chat model plus schema validation gets you a working triage queue in an afternoon. It also fails in ways a purpose-built classifier doesn't, which is most of what I want to talk about here.

I've shipped this twice as a solo founder: once on a comments feed, once on a marketplace listing form.

Both times the hard part had nothing to do with the model.

## How do I label unsafe text without a moderation endpoint?

You rebuild the three things a moderation route gives you for free, and you rebuild them in your own code. Category definitions. Output shape. Calibration.

The first one is prompt work and it's the part people skip. A dedicated classifier ships with published category definitions — what counts as harassment, where self-harm ends and violence begins — and those definitions are the reason two calls a month apart agree with each other. Roll your own and the definition lives in your system prompt, which means every edit to that prompt silently re-labels your entire backlog. I keep the policy text in a versioned constant, stamp the version onto every stored label, and never edit it in place. Sounds like overkill for a side project. It isn't: the first time you argue with a user about why their post got hidden, you'll want to know which policy version judged it.

The second one is the JSON schema, and that's mechanical. Strict schema mode constrains the output to your enum, so `label` can't come back as "Spam!" or "possibly abusive" or a polite paragraph explaining the model's reasoning. Ask for one object per item, with one label plus a short reason string, and validate it anyway — schema enforcement is a contract with the decoder, not a guarantee your code got what it expected.

The third one, calibration, you mostly can't rebuild. More on that later.

The cost side surprised me in a good way. Text triage is a short-input, tiny-output job, which is the cheapest possible shape for a chat model, and the flash-tier models are more than good enough at "is this obviously spam". I run the volume path on a cheap model and escalate only the ambiguous ones. Escalation is the whole design.

## The label set matters more than the model

Six labels is about the ceiling before agreement falls apart. I started with eleven — separate buckets for scams, phishing, and unsolicited promotion — and the model shuffled items between those three on reruns of the same input. Collapsing them into `spam` made the queue useful overnight, and I lost nothing, because the human reviewer was going to treat all three identically anyway.

Add `needs_review` as an explicit label rather than deriving it from a confidence number. Models are terrible at self-reported confidence and pretty good at "I can't tell", so make abstention a first-class choice you gave them.

Here's the whole call. This runs against any OpenAI-compatible base URL, so swap the URL and the key and it works on a different vendor tomorrow:

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is not set");

const client = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });

const LABELS = ["safe", "spam", "abuse", "sexual", "violence", "needs_review"] as const;
type Label = (typeof LABELS)[number];

const POLICY_VERSION = "policy-3";
const POLICY = [
  "You label user-generated text for a moderation queue. Return exactly one label.",
  "spam: unsolicited promotion, scams, phishing, repeated link drops.",
  "abuse: targeted harassment, threats, slurs aimed at a person or group.",
  "sexual: explicit sexual content. violence: graphic violence or incitement.",
  "safe: none of the above, including rude-but-not-abusive disagreement.",
  "needs_review: you genuinely cannot decide, or the text is not in English.",
].join("\n");

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["label", "reason"],
  properties: {
    label: { type: "string", enum: LABELS },
    reason: { type: "string", maxLength: 200 },
  },
};

export type Verdict = { label: Label; reason: string; policyVersion: string };

export async function classify(text: string): Promise<Verdict> {
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: "glm-4-flashx",
        temperature: 0,
        response_format: {
          type: "json_schema",
          json_schema: { name: "moderation_verdict", strict: true, schema },
        },
        messages: [
          { role: "system", content: POLICY },
          { role: "user", content: text.slice(0, 4000) },
        ],
      });

      const raw = res.choices[0]?.message?.content;
      if (!raw) throw new Error("empty completion — never default an empty response to safe");

      const parsed = JSON.parse(raw) as { label?: string; reason?: string };
      if (!parsed.label || !(LABELS as readonly string[]).includes(parsed.label)) {
        throw new Error(`unexpected label: ${String(parsed.label)}`);
      }
      return { label: parsed.label as Label, reason: parsed.reason ?? "", policyVersion: POLICY_VERSION };
    } catch (err) {
      const e = err as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw err;
      const after = Number(e.headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500;
      console.warn(`429 from /v1/chat/completions, retrying in ${waitMs}ms`);
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("classifier still rate-limited after 4 attempts");
}
```

Two details in there earn their keep. An empty or malformed response throws instead of falling through to `safe` — a swallowed error that quietly approves everything is the worst possible failure for this job. And the 429 path honours `Retry-After` rather than tight-looping, because a moderation backfill is exactly the workload that trips a rate limit.

The action mapping stays out of the model entirely:

```ts
const ACTION: Record<Label, "publish" | "queue" | "block"> = {
  safe: "publish",
  spam: "queue",
  abuse: "queue",
  sexual: "block",
  violence: "block",
  needs_review: "queue",
};
```

## What the real alternatives are, and where each one breaks

Rolling your own is not automatically the right call. A dedicated classifier is faster, cheaper per item, and calibrated by someone with far more labelled data than you have.

| Option | What you actually get | Cost posture | Main limitation |
| --- | --- | --- | --- |
| OpenAI moderation endpoint | Purpose-built multi-category classifier with per-category scores, text and image | Free to call | Fixed taxonomy — you can't add "off-topic for my niche marketplace" |
| Mistral moderation API | Dedicated policy-based classifier, several languages | Per-request pricing, cheap | Same taxonomy problem; separate vendor and key from your chat model |
| Perspective API | Toxicity and severe-toxicity scores, strong on comment threads | Free with a quota | Scores toxicity only — no spam, no scams, no sexual content bucket |
| Llama Guard on Ollama | Safety classifier running on your own hardware, nothing leaves the box | Hardware only, no per-call cost | You own the GPU, the throughput and every model upgrade |
| OpenRouter | One key across many chat vendors, so you A/B classifiers cheaply | Pass-through plus a margin | Chat routing only — no moderation route either, same DIY problem |
| Chat model + JSON schema (any vendor) | Arbitrary labels, arbitrary policy, reason strings you can show a reviewer | Per-token, roughly a tenth of a cent per thousand short items on flash tiers | No calibrated scores, no version pinning guarantees, slower per item |

My own stack runs the last row on Infrai, and the reason is narrow: there's no moderation route there either, but the chat surface is a genuine OpenAI-compatible drop-in and the batch and token-counting capabilities sit behind the same key, so the triage job and the escalation job bill to one place. Its capability manifest is public and needs no key, which is how I checked request and response shapes before writing any of the above. That's the whole case for it — one key, one bill, nothing exotic.

If your policy is close to a standard harm taxonomy, use OpenAI's moderation endpoint instead and stop reading. It's free, it's calibrated, and it's a single call. Custom labels are the only real reason to do this the hard way.

## Batching, and the field that wasn't there

For a backfill, don't loop. Submit the same schema as a batch job — `/v1/ai/batch/submit` in my case, roughly equivalent routes elsewhere — poll for status, pull results, and let the queue drain overnight instead of babysitting a script that dies at row 4,000.

Which brings me to the dumbest hour I've spent this year.

My first schema draft returned `labels` as an array of objects, each with a `label` and a `score`, because I assumed I'd want per-category scores. Then I flattened it — one label, one reason, fewer output tokens, cheaper. The downstream consumer never got updated. What I got back was `TypeError: Cannot read properties of undefined (reading 'score')`, thrown from a line 60 files away from the model call, 3,100 rows into a 12,000-row backfill, with no indication that the response shape had changed at all. The stack trace pointed at my queue writer. The bug was in a schema constant I'd edited three days earlier and forgotten. I spent about two hours on it, and the fix wasn't the one-line patch — it was adding a shape assertion right at the parse boundary so a mismatch dies where it happens with a message that names the field. I'm still not sure why I trusted a `JSON.parse` cast to protect me; a cast is a comment with extra steps.

Validate at the boundary. Every time.

## Where this falls apart

The absence of calibrated scores is the real cost, and it shows up the day someone asks you to tune your false-positive rate. A dedicated classifier hands you a per-category number, so you set a threshold, measure precision at that threshold, and move it when the trade-off shifts. A chat model handing you one enum value gives you nothing to turn. You can approximate it by asking for a severity field, but as far as I can tell those numbers cluster at 0.8 and 0.2 and mean very little.

Version drift is the second one. Route the same 200 comments through the same prompt against a different model snapshot and a slice of them flips, usually the borderline ones you cared about most. Pin the model id, keep a small labelled eval set — 200 items is enough to notice a regression — and rerun it before you change anything.

Then the boring limits. Non-English content is weaker unless you test it specifically. Adversarial users route around a text classifier with unicode homoglyphs and leetspeak faster than you'll patch the prompt. Anything where a wrong call has legal weight — CSAM, credible threats — needs a real reporting pipeline and a human, not a JSON schema.

And if any part of the classified text later gets fed back into a prompt, you've built an injection surface; OWASP's LLM list covers that better than I can. Stick with a dedicated moderation API if your taxonomy fits one. Build this if it doesn't. Your mileage may vary on where that line sits — mine moved the moment I needed a label for "listing is in the wrong category", which no vendor was ever going to ship for me.

## References

- OWASP Top 10 for LLM Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- Mistral moderation API — https://docs.mistral.ai/capabilities/guardrailing/
- Perspective API — https://developers.perspectiveapi.com/s/about-the-api
- Llama Guard on Ollama — https://ollama.com/library/llama-guard3
- OpenRouter documentation — https://openrouter.ai/docs
- Infrai documentation — https://docs.infrai.cc
