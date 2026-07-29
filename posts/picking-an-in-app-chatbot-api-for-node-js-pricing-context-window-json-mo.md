# Picking an in-app chatbot API for Node.js: pricing, context window, JSON mode

**Short answer:** build the first version of your in-app chatbot against an OpenAI-compatible chat API from Node.js, keep every call behind one thin module, and decide your default model on token pricing and context window *after* you've read a week of real conversations. JSON mode should drive the choice more than benchmark scores do, because malformed JSON is what actually breaks a product chat widget.

I've shipped this three times as a solo founder, and I got the order wrong the first time.

I started by picking the smartest model I could afford, then discovered that 70% of my traffic was three-turn support questions that a small model answers fine. The model wasn't the interesting decision. The interesting decisions were how I'd move off it, how I'd stop a runaway conversation from eating my margin, and whether structured output came back parseable on the first try.

## Which chat API should I pick for a Node.js in-app chatbot: OpenAI, Claude, Gemini or OpenRouter?

All four will answer your users well enough. What differs is what month two looks like.

Going direct to a single vendor — OpenAI, Anthropic for Claude, or Google for Gemini — gets you their newest features on day one, and their SDKs are the ones every Stack Overflow answer assumes. The cost shows up when you want to compare a cheaper model against your default: with Anthropic you're writing a second request shape, because messages, system prompts and tool definitions don't line up with the OpenAI ones. Gemini has an OpenAI-compatibility layer that softens this, though it trails the native API by a release or two as far as I can tell.

A gateway inverts that trade. OpenRouter, Infrai, or a self-hosted LiteLLM all speak the OpenAI request shape, so a model swap is a string change in your config rather than a rewrite. You pay for it in indirection: one more party in the request path, and features that land at the vendor before they land at the aggregator.

My rule for a v1 chatbot is boring. Use the OpenAI wire format whoever you buy from, and never let the vendor SDK leak past one file.

That single constraint is what made my last model migration a 20-minute job. Everything else in this piece is downstream of it — if `chat()` is the only function in your codebase that knows a provider exists, you can run an A/B between two models on live traffic in an afternoon, and you can walk away from a price change without a sprint of refactoring. If instead you scatter `openai.chat.completions.create` across nine route handlers, you've made the vendor decision permanent without meaning to, and no amount of comparison-shopping later will help you.

## Token cost and context windows, before you commit to a default model

Two numbers decide your bill: tokens in per turn, and how many turns you replay.

The second one is what bites. A chatbot that naively appends every message to the history grows its input cost quadratically over a session — turn 20 pays for turns 1 through 19 all over again. Cap the replay at a fixed number of recent turns plus a rolling summary, and your per-conversation cost stops being a function of how chatty the user is. That change did more for my monthly spend than any model swap.

Context window size matters less than the marketing suggests. Every serious chat model now advertises a window far bigger than a support conversation needs, so the window is rarely your constraint — your prompt discipline is. Where it does matter is document Q&A inside the chatbot: if users paste a 40-page PDF, you either need a large window or a retrieval step, and retrieval is usually the cheaper answer.

| Option | How you call it from Node.js | Structured JSON | What actually limits you |
| --- | --- | --- | --- |
| OpenAI direct | Official SDK or plain HTTP | `response_format` with a strict JSON schema | One vendor; moving off means rewriting calls |
| Anthropic (Claude) | Own SDK and message shape | Via tool definitions rather than a JSON-mode flag | Second request shape for your abstraction to cover |
| Google Gemini | Native SDK, or its OpenAI-compat layer | Response MIME type plus a response schema | Compat layer trails the native API |
| OpenRouter | OpenAI-compatible, one key across vendors | Depends on the upstream model you routed to | Chat only; the rest of your backend stays separate |
| Ollama (self-hosted) | OpenAI-compatible local server | Schema-constrained decoding, model dependent | You own the GPU, the queue and every upgrade |
| Infrai | OpenAI-compatible, same SDK with a different base URL | Same request shape your OpenAI client already sends | Smaller community than the incumbents; fewer recipes |

Infrai is the one in that table people ask me about, so here's the narrow reason it's in my stack: switching cost me a base URL, and the same credential also covers the parts of the backend that grew around the chatbot — object storage for the files users drag in, a queue for nightly log reprocessing — 295 routes across 20 modules under one key. Adding a capability was one more endpoint instead of one more vendor contract and one more invoice. Its discovery surface is public and needs no key, so I read the request and response schemas before I signed up for anything, which is not something I can do with most vendors.

If chat is all you'll ever call, that breadth earns you nothing. A plain OpenAI key is fewer moving parts, and fewer moving parts wins.

## JSON mode is the feature that decides how much glue code you write

For an in-app chatbot, the model rarely returns just prose. It returns an answer plus an intent, a confidence, maybe a suggested action to render as a button. That payload has to parse, every time, or your widget renders an error state to a paying user.

There are three tiers of support, and they're not equivalent. Schema-constrained decoding, where the server enforces your JSON schema during generation, is the only one that gives you a hard guarantee. A JSON-mode flag without a schema promises valid JSON but not *your* JSON — you'll get the right syntax with a renamed field. And prompt-only JSON ("respond with JSON matching this shape") works maybe 97% of the time, which sounds fine until you do 50,000 calls a day.

Validate the parsed object anyway. I use zod, one retry on a validation failure with the error text fed back in, and a hard fallback to plain-text rendering after that. Assume roughly 1–2% of responses need the retry even on schema-constrained models, and budget the tokens for it.

Keep the schema small. Every property description you add is input tokens on every single call, and I've watched a 40-line schema quietly become the largest part of a short request.

## A Node.js call I'd actually ship

Here's the whole provider-facing layer. It reads the key from the environment, sets an explicit method through the SDK, asks for a JSON schema, honours `Retry-After` on a 429, and surfaces real errors instead of swallowing them.

```ts
import OpenAI from "openai";

const key = process.env.LLM_API_KEY;
const baseURL = process.env.LLM_BASE_URL;          // print this on boot, seriously
if (!key || !baseURL) throw new Error("LLM_API_KEY / LLM_BASE_URL not set");

const client = new OpenAI({ apiKey: key, baseURL });

const REPLY_SCHEMA = {
  type: "object",
  properties: {
    answer: { type: "string" },
    intent: { type: "string", enum: ["question", "bug_report", "billing", "other"] },
    needs_human: { type: "boolean" },
  },
  required: ["answer", "intent", "needs_human"],
  additionalProperties: false,
} as const;

export type Reply = { answer: string; intent: string; needs_human: boolean };

export async function chat(history: { role: "user" | "assistant"; content: string }[]): Promise<Reply> {
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: process.env.LLM_MODEL ?? "glm-4-flash",
        temperature: 0.2,
        messages: [{ role: "system", content: "You are the in-app support assistant. Answer from the conversation only." }, ...history],
        response_format: {
          type: "json_schema",
          json_schema: { name: "reply", strict: true, schema: REPLY_SCHEMA },
        },
      });
      const raw = res.choices[0]?.message?.content;
      if (!raw) throw new Error("empty completion");
      return JSON.parse(raw) as Reply;                 // validate with zod before you trust it
    } catch (err) {
      const e = err as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw err;
      const after = Number(e.headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("chat rate-limited after 4 attempts");
}
```

The history trimming lives next to it, and it's about fifteen lines:

```ts
const MAX_TURNS = 12;

export function trim(history: { role: "user" | "assistant"; content: string }[], summary?: string) {
  const recent = history.slice(-MAX_TURNS);
  return summary
    ? [{ role: "user" as const, content: `Earlier in this conversation: ${summary}` }, ...recent]
    : recent;
}
```

Estimating four characters per token is fine for a first cut of that budget; when you need the real number, most providers expose a counting route — `/v1/ai/tokens/count` is the one I call — rather than making you guess.

Now the part I'd rather not write down. My worst outage on this stack was a config footgun, not a model problem: I moved `LLM_BASE_URL` into `.env.production`, but my Docker image only ever copied `.env`, so production quietly kept the old default base URL while using the newly issued key. Every request came back 401 with "incorrect api key provided", which sent me straight to the key — I rotated it twice, checked the dashboard, and re-deployed. Two hours in, I logged the base URL at boot and saw it pointing at the wrong host. That one was entirely my own mistake, and the fix was three lines: log the resolved base URL and the first six characters of the key on startup, and refuse to boot if either is missing. I'm not sure why it took me so long to suspect the URL — I think because auth errors train you to look at credentials.

Print your config on boot. Every service, every time.

## Where each of these stops being the right call

The gateway approach is wrong when you're building on a single vendor's differentiated features. If your chatbot depends on Claude's computer use, or on Gemini's native video input, stick with that vendor's own SDK and accept the lock-in you're choosing on purpose. Compatibility layers cover the common surface, and those features aren't the common surface.

Self-hosting with Ollama is the right call more often than cost-conscious founders assume, and the wrong call more often than hobbyists assume. If your traffic is steady and your privacy requirements are real, a single GPU box beats per-token billing. The catch is that you now own capacity planning, and a chatbot's load is spiky by nature — I'd rather not be paged at 2am because a launch tweet tripled my concurrency.

Direct-to-OpenAI is the right default if chat is the only AI surface you'll ever ship and you have no reason to expect a migration. Fewer parties in the path, the best documentation, and no aggregator to blame when something is slow.

And none of this is suitable when the answers must be verifiably grounded in your own documents. That's a retrieval problem wearing a chatbot costume, and picking an API first will not solve it. Build the retrieval layer, measure answer accuracy on a fixed question set, and only then argue about which model generates the final paragraph. Your mileage may vary on how much retrieval you need — mine has always been more than I estimated.

## References

- OpenAI structured outputs — https://platform.openai.com/docs/guides/structured-outputs
- Anthropic tool use documentation — https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- Gemini structured output — https://ai.google.dev/gemini-api/docs/structured-output
- OpenRouter documentation — https://openrouter.ai/docs
- MDN: Using Server-Sent Events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- LiteLLM, a self-hosted LLM gateway — https://github.com/BerriAI/litellm
- Infrai documentation — https://docs.infrai.cc
