# How to compare text-to-image API cost per image for a startup MVP in Node.js

Bottom line: choose a text-to-image API by measuring cost per *accepted* image on your own prompts, not by lining up the per-image list price from each pricing page. For a startup MVP the gap between those two numbers is mostly rejected renders, uncapped retries and a resolution default nobody looked at — in my case the effective cost landed about four times above what I'd penciled in. A fifty-prompt harness in Node.js takes an afternoon and settles the question with your data instead of someone else's benchmark.

The data flow is boring, which is exactly why the estimates go wrong. Your server sends a prompt string over HTTPS, waits somewhere between two and forty seconds, gets back image bytes or a signed URL, and writes the result to object storage. One call, one charge, done.

Everything that inflates the bill happens around that call.

## What does a text-to-image API actually cost per image at MVP scale?

Three multipliers sit between the list price and your invoice, and only one of them is on the vendor's website.

The first is acceptance rate. If a product surface shows the user one picture and 40% of the time they hit regenerate, your real unit is two renders, not one. Acceptance is a function of your prompt template far more than of the model, and it's the number I'd measure first because it's also the one you can move. Tightening a prompt template from a loose one-liner to a structured description with negative constraints took my regenerate rate from roughly one in three down to one in six, and that single change did more for the bill than any provider switch I tried afterwards.

The second is retry amplification. Image generation is slow enough that HTTP timeouts get hit constantly, and a naive wrapper that retries on timeout will happily pay for renders that already completed on the far side. A 429 is free — you were rejected before any GPU ran. A client-side timeout on a request that succeeded server-side is not free, and it's invisible in your own logs because your process never saw a response.

The third is resolution and step count. Providers that bill per megapixel or per second of compute will charge you two to four times more for a 2048px render than a 1024px one, and most SDK defaults are generous. Nobody bills you for the images your users liked. They bill you for the ones you rendered.

So the metric worth optimizing is dollars per accepted image, with p95 latency as a hard secondary constraint, since a text-to-image call that takes 30 seconds changes your whole UX and pushes you toward a job queue.

## Measuring cost per accepted image in Node.js

Here's the harness I keep reusing. It's deliberately thin — one file, no dependencies beyond the runtime's own fetch, and a cost function you fill in by hand from each provider's own pricing page.

```ts
// image-cost-probe.ts — measure cost per ACCEPTED image, not per API call.
type Provider = {
  name: string;
  endpoint: string;         // the provider's text-to-image endpoint
  apiKey: string;
  costPerRender: number;    // USD for one render at the size you actually ship
};

type Attempt = { ok: boolean; ms: number; usd: number; renders: number };

const MAX_ATTEMPTS = 3;     // hard cap; uncapped retries are how a bill runs away
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function render(p: Provider, prompt: string): Promise<Attempt> {
  const started = Date.now();
  let usd = 0;
  let renders = 0;

  for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
    const res = await fetch(p.endpoint, {
      method: "POST",
      headers: { "content-type": "application/json", authorization: `Bearer ${p.apiKey}` },
      body: JSON.stringify({ prompt, width: 1024, height: 1024 }),
    });

    // Rate limiting is free: nothing was rendered, so nothing is billed. Back off and retry.
    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? attempt * 2);
      await sleep(retryAfter * 1000);
      continue;
    }

    renders += 1;
    usd += p.costPerRender;   // anything past the queue is billable, success or not
    if (!res.ok) return { ok: false, ms: Date.now() - started, usd, renders };

    await res.json();
    return { ok: true, ms: Date.now() - started, usd, renders };
  }

  return { ok: false, ms: Date.now() - started, usd, renders };
}

export async function benchmark(p: Provider, prompts: string[], accepted: (a: Attempt) => boolean) {
  const runs: Attempt[] = [];
  for (const prompt of prompts) runs.push(await render(p, prompt));

  const usd = runs.reduce((n, r) => n + r.usd, 0);
  const kept = runs.filter((r) => r.ok && accepted(r)).length;
  const sorted = runs.map((r) => r.ms).sort((a, b) => a - b);

  return {
    provider: p.name,
    billedRenders: runs.reduce((n, r) => n + r.renders, 0),
    accepted: kept,
    usdPerAccepted: kept ? usd / kept : Infinity,
    p95ms: sorted[Math.floor(sorted.length * 0.95)] ?? 0,
  };
}
```

The `accepted` callback is where the judgment lives, and I don't think there's a clean way to automate it at MVP scale. I've done it by dumping the grid into a static HTML page and clicking through fifty images myself — twenty minutes, and the number you get is honest. Automated scoring with a vision model is possible; I'm not sure it's worth the extra moving part before you have real users.

Run the same prompt set against every candidate on the same day. Providers tune models continuously, so a comparison you ran three months ago is a historical artifact, not a decision input.

## Pricing shapes, and where the bill actually comes from

Providers don't just differ on price, they differ on what the price is attached to, and that shape decides which of your engineering choices show up on the invoice.

| Billing shape | You pay for | Hidden driver | Reasonable fit |
|---|---|---|---|
| Flat per image | one render, fixed size | rejected renders and retries | predictable MVP volume, easy forecasting |
| Per megapixel or per step | resolution and sampling steps | defaults set too high | tuning quality down to control spend |
| Per second of compute | wall-clock on a hosted model | cold starts and queue time | bursty traffic, self-tuned pipelines |
| Rented GPU, self-hosted weights | the instance, always on | idle hours between requests | steady high volume, strict data residency |

The names that come up in most of these comparisons — OpenAI's image endpoint, Stability AI, Ideogram, fal.ai — sit across at least three of those four rows, which is why a straight per-image price comparison between them is close to meaningless. A flat per-image charge and a per-second charge only become comparable after you've fixed the resolution, the step count, and the acceptance bar.

The catch with the cheapest row on that table is real, and worth stating plainly: renting a GPU and running open weights yourself has an excellent marginal cost and a terrible fixed cost. Below roughly a few thousand images a month, an idle instance costs more than the API calls you avoided, and you've also bought yourself model updates, driver pinning and a capacity page. Stick with a hosted API until your volume is boringly predictable. In the other direction, if you have data-residency obligations or you need a fine-tuned model that hosted providers don't support, the hosted route is closed to you regardless of the arithmetic and the fixed cost stops being a trade-off at all.

One more constraint that doesn't show up in pricing tables: rate limits on a new account are usually low, and the ceiling matters more than the unit price if your MVP has any launch-day spike. Check the documented concurrency limit before you commit.

## The month my image bill hit $487

I'd budgeted $50 a month. A small product, maybe 200 active users, each generating a handful of images — the arithmetic looked fine on the back of an envelope.

The first full month came back at $487, and it took me most of a morning to work out why. Three causes, all mine. The regenerate button had no per-user cap, so a dozen users discovered they could refine a prompt indefinitely and one of them ran 340 renders in a single evening. My retry wrapper treated a client-side timeout as transient and retried up to five times, and because the renders were completing server-side at around 24 seconds against a 20-second timeout, I was paying for three or four copies of images my process never even received. And the frontend was requesting 1536px because that's what I'd copied out of a quickstart eight weeks earlier, on a provider that bills per megapixel.

Fixing the timeout alone removed most of it. The lesson I'd generalize: instrument billable renders as a counter in your own telemetry, separate from successful responses, because those two numbers diverging is the whole failure mode.

## Keeping the bill boring after launch

The operational work here is small but it has to exist before you ship, not after the first surprise invoice. Emit a counter for every request that made it past the queue, tagged by provider and by user, and alarm when the daily total crosses about twice your forecast — that alarm has paid for itself twice for me. Set your HTTP timeout well above the provider's own p99, not below it, because a timeout that fires on a healthy request is a charge with nothing to show for it. Cap regenerations per user per day in the product, since a quota is easier to explain to a customer than an unexpected bill is to explain to yourself.

Keep the provider call behind one thin interface with the prompt, size and seed as its only arguments, so swapping vendors is an afternoon rather than a refactor. If you'd rather not maintain that shim, an open-source gateway like LiteLLM already normalizes multi-provider routing, retries and per-key spend tracking, though its coverage is centered on text models and you should verify image support against the current source before you depend on it.

Re-run the fifty-prompt harness quarterly. Prices move, models move more, and your prompt templates will have drifted — the answer you got in 2026 has a shelf life measured in months. Your mileage may vary on the exact numbers, but the method holds: measure acceptance, count billable renders, and compare on dollars per accepted image.

## References

- RFC 6585, Additional HTTP Status Codes (429 Too Many Requests): https://www.rfc-editor.org/rfc/rfc6585#section-4
- MDN, Retry-After HTTP header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After
- LiteLLM, open-source multi-provider gateway with spend tracking: https://github.com/BerriAI/litellm
- OpenTelemetry semantic conventions for generative AI: https://opentelemetry.io/docs/specs/semconv/gen-ai/
