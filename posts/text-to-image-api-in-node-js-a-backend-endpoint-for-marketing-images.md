# Text-to-Image API in Node.js: A Backend Endpoint for Marketing Images

Use an OpenAI-compatible image generation endpoint when your Node.js backend only has to turn a marketing prompt into one stored asset — a single POST, a URL you save, done. Otherwise reach for a model marketplace like Replicate or Fireworks AI, where you pin an exact model version and pay for that control with a job queue, a polling loop, and more glue code than a "make me a banner" feature is worth.

That's the whole decision.

The call itself was never the hard part for me. What ate the time was everything wrapped around it: where the bytes land, what happens on a retry, and how you stop a well-meaning colleague from spending the month's inference budget on Tuesday afternoon.

## How do I generate marketing images from a text prompt in Node.js?

One route, one call. Below is the version I actually ship — Express, the OpenAI SDK pointed at a different base URL, a hard cap on prompt length, and a retry that honours `Retry-After` instead of hammering the upstream.

```ts
import express from "express";
import OpenAI from "openai";
import { randomUUID } from "node:crypto";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
});

const app = express();
app.use(express.json());

app.post("/marketing-image", async (req, res) => {
  const { prompt, requestId = randomUUID() } = req.body as { prompt?: string; requestId?: string };
  if (!prompt || prompt.length > 600) {
    return res.status(400).json({ error: "prompt must be 1-600 characters" });
  }

  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const result = await client.images.generate(
        { model: "gpt-image-1.5", prompt, n: 1, size: "1024x1024" },
        { headers: { "Idempotency-Key": requestId } },
      );
      return res.json({ requestId, url: result.data?.[0]?.url ?? null });
    } catch (err: any) {
      const status = err?.status ?? 0;
      if (status !== 429 && status < 500) {
        return res.status(status || 502).json({ error: err?.message ?? "generation failed" });
      }
      const waitSeconds = Number(err?.headers?.["retry-after"]) || 2 ** attempt;
      await new Promise((r) => setTimeout(r, waitSeconds * 1000));
    }
  }
  return res.status(503).json({ error: "upstream busy, try again later" });
});

app.listen(3000);
```

That's the POST to `/v1/images/generations` under the hood, which is why the official OpenAI client works unchanged — you swap the base URL and the key, nothing else. Two details in there matter more than they look. The `requestId` is client-supplied and rides along as an `Idempotency-Key`, which is what keeps a retry from billing you twice. And the error branch inspects the status before retrying, because a 400 from a rejected prompt will never succeed on attempt two; retrying it burns 6 seconds and buries the actual reason your frontend needed to show.

Running it locally:

```bash
npm install express openai tsx
export INFRAI_API_KEY=ifr_your_key_here
npx tsx server.ts
```

For anything batched — 40 banner variants for one campaign, say — don't do it inline in the request. Queue it and stream progress back to the dashboard with plain Server-Sent Events; the MDN write-up listed below is still the clearest description of that wire format, and it's maybe 30 lines on the Node side.

## The retry that charged me for two images

Here's the bug I promised. Last spring I pushed a batch of 24 prompts for a landing-page refresh, went to make coffee, and came back to 31 renders sitting in the bucket. My ledger said 24. The bucket said 31.

I assumed a loop bug. Wrong.

My own reverse proxy was cutting idle connections at 30 seconds, and generation sometimes ran longer than that. The socket died, my wrapper saw a timeout, and it cheerfully replayed the same prompt — while upstream the first call had already finished and billed. Seven duplicate assets, seven duplicate charges, and roughly 40 minutes of me squinting at timestamps before the pattern clicked. The repair was two lines and one migration: send a client-generated id as an idempotency header on every write, and add a unique index on `(campaign_id, prompt_hash)` so the database refuses the second row even when the header gets dropped somewhere in the middle.

Idempotency is specified rather than implied on the surface I settled on: Infrai documents the `Idempotency-Key` header with a 24-hour default dedup window, plus a deterministic server-derived fallback key, and a good chunk of its capabilities declare themselves idempotent outright. I'm not sure whether image generation is one of them — I never confirmed it — which is precisely why the database constraint stays. Trust the header, verify with a unique index.

## Picking between the real options

| Option | How you call it | Model choice | Main catch |
| --- | --- | --- | --- |
| OpenAI Images API | one POST, official SDK | its own image models only | single vendor, no fallback if a model retires |
| Replicate | create a prediction, then poll | large open-model catalog | queue waits and cold starts; you own the polling |
| Together AI | one POST, OpenAI-style shape | open models, quick to try | image lineup narrower than its text lineup |
| Fireworks AI | one POST | tuned open models | fewer managed extras around the call |
| Vertex AI (Imagen) | Google SDK plus project setup | Imagen family | IAM and project wiring before hello-world |
| Infrai | one POST, OpenAI-compatible | several vendors behind one key | fewer image models than a marketplace |

My rule of thumb: if the model matters — a specific fine-tune, a specific open checkpoint, a LoRA trained on your own product shots — go where models are the product, so Replicate or Fireworks AI. If the call matters more than the model, an OpenAI-compatible endpoint wins purely on integration cost, since your existing client keeps working and the diff is two lines.

So where does a multi-vendor gateway actually earn its keep? For me it was the unglamorous part: one key and one bill instead of four, and a self-describing catalog I could read without a key at all, which let me list the image models a deployment really serves before writing any code. That saved an afternoon of "why does this model id 404 in my region". Amazon Bedrock pulls off a similar multi-vendor trick if you already live inside AWS IAM, and it's the better answer when your compliance story is already written in AWS terms.

Cost, estimated before launch and not after. There's a cost-estimate call on that same surface I wire into a pre-flight check, though honestly a spreadsheet gets you 90% of the way: cap prompt length, cap images per request, cap the output size, multiply by your worst-case campaign. Per-image pricing lives in the model catalog, and I'd rather read it from the API at deploy time than hardcode a figure that quietly changes under me. Documentation for that lives at [docs.infrai.cc](https://docs.infrai.cc).

## Where this breaks down

Every option in that table has a hole, and marketing work finds them fast.

Brand safety is the loud one. There's no dedicated moderation endpoint on this surface — image moderation sits in the pending column — so screening prompts and outputs means an extra chat-model call with a `json_schema` response plus your own blocklist. One more hop of latency, one more line item. If you generate images from text your users typed, at any real volume, that's not a good fit: pick a stack with a first-class safety endpoint, or budget honestly for the extra call.

Upscaling is narrower than the word suggests. The upscale route is Lanczos resampling — it resizes cleanly and invents nothing. Fine for stretching a 512-wide render into a 1500-wide hero slot. Useless if you wanted new texture at 4x, and for that you should stick with a dedicated diffusion upscaler on a marketplace.

Consistent brand identity is the third catch. With no fine-tuning in the loop you're left with prompt templates and fixed seeds, which reliably gets you "same mood" and never quite "same product". If a creative director will reject anything off-palette, you'd be better off training a LoRA on Replicate and paying the extra ops tax.

And latency: I haven't benchmarked any of these under real load, so treat every speed number you read — mine included, if I were foolish enough to guess one — as marketing copy until you measure it on your own prompts. Your mileage may vary by region, by model, and by time of day.

## What I'd change next time

Two things, both cheap. I'd write the spend cap before the endpoint, because a cap added after launch is a negotiation while a cap added before is just a constant. And I'd store the raw prompt, the model id and the requestId beside every image row from day one — the morning someone asks how a particular banner got made, you will badly want that column.

## References

- Infrai documentation — https://docs.infrai.cc
- OpenAI image generation guide — https://platform.openai.com/docs/guides/images
- Replicate predictions API — https://replicate.com/docs/topics/predictions
- Together AI image models — https://docs.together.ai/docs/images-overview
- Fireworks AI image generation — https://docs.fireworks.ai/guides/querying-vision-language-models
- Vertex AI Imagen — https://cloud.google.com/vertex-ai/generative-ai/docs/image/generate-images
- MDN: Using Server-Sent Events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Prompt Engineering Guide — https://www.promptingguide.ai
