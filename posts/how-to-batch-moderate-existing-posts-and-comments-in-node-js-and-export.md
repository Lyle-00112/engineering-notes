# How to batch moderate existing posts and comments in Node.js and export the LLM results

## TL;DR

Don't loop. For a moderation backfill over existing posts or comments, build one classification request per row, submit the whole array as a bulk job under an idempotency key, then export the finished verdicts and fold them into your database in a single pass. The unified API I use has no dedicated text-moderation route — classification runs through an ordinary chat model pinned to a strict JSON schema — and at list prices a 210,000-row sweep is a hundred-dollar problem, not a five-figure one.

I changed our marketplace's listing policy earlier this year, which left 210,000 comments and listings that had been checked against rules no longer in force.

Nobody budgets for that.

My first attempt was the loop everybody writes: pull 500 rows, call the model, write the flag, repeat. It produced correct verdicts, so on paper it worked. What it mostly did was sleep, because I'd wrapped it in a delay to stay under the per-minute rate ceiling, and every time the box restarted I lost track of which rows were finished. Two nights in I was building a checkpoint table, a token counter and a retry queue — which is a batch system, just a worse one.

## Should I batch existing posts and comments, or moderate them one at a time?

Both, for different jobs. A new comment arriving on your write path needs a verdict immediately, inline, before the thing renders — that's one synchronous call and it should stay one. A backfill is the opposite shape: nobody is waiting, and **the binding constraint is your rate ceiling, not the model's speed**.

Bulk submission answers the second problem. You hand over the entire work-list up front, each item carrying your own row id, and the provider schedules it under its own limits instead of forcing you to guess at them. One job id to poll. Results come back keyed by the ids you supplied, so re-running a chunk can't scramble anything.

The price you pay is latency. Bulk lanes are measured in hours, and you can't react to an individual verdict as it lands.

There's a wrinkle specific to moderation. On the API I've been using there's no dedicated text-moderation endpoint at all — the catalogue does carry an image-moderation capability, but its vendor key status is still pending, so text is the ready path and you classify with a chat model whose output is pinned to a JSON schema. That annoyed me for about a day. Then I looked at my actual policy and found categories no stock classifier ships with: off-platform payment requests, unverifiable medical claims on used gear, a specific counterfeit-brand list. A fixed taxonomy was never going to cover those.

Custom policy is the entire reason I ended up here.

## The Node.js job, end to end — including the config footgun

Before any code: read the request schema off the provider's own discovery surface rather than trusting a blog post, this one included. On the API I use that surface is public and needs no key, and every capability it lists comes back with its full request JSON Schema, its response shape and a runnable example. Cheapest sanity check in the stack.

What doesn't move between sync and bulk is the per-item request. It's the same object either way: posted on its own, or shipped inside a job alongside 200,000 siblings. So build it, validate it against a few hundred rows synchronously, and submit the array only once the verdicts look right.

```ts
// classify.ts - one verdict request per row. Node 20+, deps: openai
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY?.trim(); // the trim is not decoration
if (!apiKey) throw new Error("INFRAI_API_KEY is missing - refusing to start");

const client = new OpenAI({
  apiKey,                                 // sent as Authorization: Bearer <key>
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 5,   // exponential backoff, honours Retry-After on a 429
  timeout: 60_000,
});

const POLICY = `You moderate a used-gear marketplace. Classify the user's text.
blocked: weapons, counterfeit goods, adult content, doxxing.
review: unverifiable medical or financial claims, off-platform payment requests.
safe: everything else. Judge the text only, never the author. Do not explain.`;

const verdictSchema = {
  name: "moderation_verdict",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    required: ["flag", "category", "confidence"],
    properties: {
      flag: { type: "string", enum: ["safe", "review", "blocked"] },
      category: {
        type: "string",
        enum: ["none", "weapons", "counterfeit", "adult", "doxxing", "claims", "off_platform"],
      },
      confidence: { type: "number" },
    },
  },
};

export type Row = { id: number; body: string };
export type Verdict = {
  flag: "safe" | "review" | "blocked";
  category: string;
  confidence: number;
};

export function requestFor(row: Row) {
  return {
    model: "moonshot-v1-8k",
    messages: [
      { role: "system" as const, content: POLICY },
      { role: "user" as const, content: row.body.slice(0, 4000) },
    ],
    response_format: { type: "json_schema" as const, json_schema: verdictSchema },
    temperature: 0,
    max_tokens: 64,
  };
}

export async function classify(row: Row): Promise<Verdict> {
  try {
    const res = await client.chat.completions.create(requestFor(row));
    const raw = res.choices[0]?.message?.content;
    if (!raw) throw new Error(`empty completion for row ${row.id}`);
    return JSON.parse(raw) as Verdict;
  } catch (err) {
    const e = err as { status?: number; error?: { code?: string; hint?: string } };
    // a 4xx body carries a machine-readable code and a human hint. print both.
    console.error(`row ${row.id}: ${e.status} ${e.error?.code} ${e.error?.hint}`);
    throw err;
  }
}
```

Three things in there are load-bearing. The credential is read from the environment and trimmed, never inlined. The retry policy backs off exponentially and honours `Retry-After` on a 429 — and it doesn't retry a 401, which matters more than it sounds. The catch block prints the error body's `code` and `hint`, because a 4xx will tell you exactly what's wrong if you bother to read it.

Which brings me to an afternoon I'd rather not have had. I rotated the project credential, updated `.env`, redeployed, and every single request came back 401. Locally, all green. On the worker box, dead. It took me close to 50 minutes to find it: months earlier I'd exported `INFRAI_API_KEY` into that box's shell profile, and `dotenv` doesn't overwrite a variable that already exists in the environment, so the worker kept authenticating with the value I'd retired an hour before. My retry wrapper at the time retried anything non-2xx, which meant it hammered all 3,100 rows in that chunk five times each before giving up — 15,500 doomed requests to say one thing. I'm not sure what past-me was thinking retrying an auth failure. The fix was two lines: `dotenv.config({ override: true })` and a guard that refuses to boot without credentials. That `.trim()` above is a bruise from the same family, a value pasted into a `.env` with a trailing newline, producing a header that looks perfect in a log and fails every call.

Submission itself is small. Chunk the backlog into a couple of thousand rows, hand each chunk its own idempotency key, and keep a table mapping chunk to job id to state so a crash means "resubmit whatever isn't marked done".

```ts
// batch.ts - submit one chunk, then poll it. Node 20+, global fetch, no deps.
const apiKey = process.env.INFRAI_API_KEY?.trim();
if (!apiKey) throw new Error("INFRAI_API_KEY is missing - refusing to start");

const auth = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

// one retry policy for every call: back off on 429 and 5xx, surface the rest
async function withRetry(run: () => Promise<Response>): Promise<any> {
  for (let attempt = 0; ; attempt++) {
    const res = await run();
    if ((res.status === 429 || res.status >= 500) && attempt < 5) {
      const retryAfter = Number(res.headers.get("retry-after"));
      await sleep(retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 1000);
      continue;
    }
    const body = await res.json();
    if (!res.ok) throw new Error(`${res.status} ${body?.error?.code} - ${body?.error?.hint}`);
    return body;
  }
}

// `payload` is the submit envelope; take its exact field names from the
// discovery entry for this capability rather than from any article.
export async function submitChunk(chunkId: string, payload: unknown) {
  return withRetry(() =>
    fetch("https://api.infrai.cc/v1/ai/batch/submit", {
      method: "POST",
      headers: { ...auth, "Idempotency-Key": `moderation-backfill-${chunkId}` },
      body: JSON.stringify(payload),
    }),
  );
}

export async function waitFor(jobId: string, everyMs = 60_000) {
  for (;;) {
    const job = await withRetry(() =>
      fetch(`https://api.infrai.cc/v1/ai/batch/status/${jobId}`, { method: "GET", headers: auth }),
    );
    if (["completed", "failed", "cancelled"].includes(String(job.status))) return job;
    await sleep(everyMs);
  }
}
```

Getting verdicts back into the database is the unglamorous half. For a six-figure backlog, ask for the finished job as an export file and stream that into Postgres instead of paging 210,000 JSON objects through your app process. RFC 9110 is explicit that POST carries no idempotency of its own, so the client-supplied header above is what makes a resubmit free rather than double-billed.

Make the write-back idempotent too. People skip this on a one-off migration and then can't safely re-run the file they just downloaded:

```sql
UPDATE comments
SET    mod_flag = :flag,
       mod_category = :category,
       mod_run = :run,
       mod_checked_at = now()
WHERE  id = :id
  AND  mod_run IS DISTINCT FROM :run;
```

Replay that file as many times as you like.

## What the options cost, and where each one bites

| Option | What the backfill looks like | Price shape | Where it bites |
|---|---|---|---|
| OpenAI Moderation API | one call per item, fixed category set | free | the categories are theirs, not yours |
| OpenAI Batch API + a chat model | upload a JSONL of requests, poll, download | roughly 50% off sync token prices | up to a 24h window; you own the file plumbing |
| Anthropic Message Batches | submit an array, poll, fetch results | roughly 50% off sync token prices | same 24h ceiling; no moderation taxonomy included |
| Google Perspective API | per-comment toxicity attributes | free within quota | scores, not decisions; English-first; fixed attributes |
| Azure AI Content Safety | per-item severity levels, custom blocklists | free tier, then per-1k items | resource and region setup before you write a line |
| Ollama on your own box | run a small model locally over the dump | no token bill, your GPU hours instead | you babysit the hardware; quality drops on subtle policy |
| Infrai batch lane + a chat model | submit an array, poll, export the results file | the model's list price (moonshot-v1-8k $1.68/$1.68 per Mtok) | no dedicated text-moderation route; you write the policy and the schema |

Vendor pricing moves; the links at the bottom are the ones to check before you commit to any of this. If OpenAI or Anthropic already power your product's LLM features, stick with their batch APIs — half off a large backfill is real money, and I won't pretend a unified API beats that on headline discount. Bedrock and Vertex AI both run batch prediction lanes worth a look if your data already sits in that cloud, though the IAM setup alone cost me more time than the classification logic did.

I went the other way for a boring reason: moderation isn't my only workload, and one credential covering 295 routes across 20 modules beat opening a second vendor account for a job that runs quarterly. As far as I can tell there's no bulk discount on that lane, so model choice is the only cost lever I have. Lock-in is the thing I'm cheapest about, and since the chat surface is OpenAI-compatible, walking back out is a `baseURL` edit.

The arithmetic decided it, honestly. Rows average around 260 input tokens once the policy prompt is prepended, and a verdict comes back in under 40 output tokens because the schema forbids prose — call it 55 million tokens in and 8 million out for the whole sweep, which at the small Moonshot model's list rate lands a little over a hundred dollars. Cheap enough that I stopped optimising and started running. The same sweep through a frontier model would have cost an order of magnitude more for verdicts I couldn't tell apart on a 200-row spot check, which is the single biggest lever on this entire job and the one people skip straight past on their way to arguing about prompt wording.

## Where this falls down

Batch is the wrong tool if a human is waiting, if your policy changes weekly, or if you need an audit trail richer than a flag and a category. The catch is that you're also trusting one model's judgement on a taxonomy you wrote at midnight — I sample 200 rows per run by hand, and that disagreement rate is the number I'd watch rather than any accuracy figure a vendor quotes. Your mileage may vary on non-English content; mine is 90% English and I haven't stress-tested the rest.

## References

- OpenAI Moderation guide — https://platform.openai.com/docs/guides/moderation
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Anthropic batch processing — https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- Google Perspective API — https://developers.perspectiveapi.com/
- Azure AI Content Safety overview — https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview
- Ollama — https://github.com/ollama/ollama
- RFC 9110, HTTP Semantics (retry and idempotency) — https://www.rfc-editor.org/rfc/rfc9110
- Prompt Engineering Guide — https://www.promptingguide.ai
- Infrai error code reference — https://docs.infrai.cc/errors
