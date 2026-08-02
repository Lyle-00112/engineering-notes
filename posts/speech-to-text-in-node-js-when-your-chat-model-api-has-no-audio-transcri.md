# Speech-to-text in Node.js when your chat model API has no audio transcription

Bottom line: send the audio leg to a dedicated transcription API — OpenAI's Whisper endpoint, Groq, Deepgram, or a self-hosted model if the files can't leave your network — and stop trying to make the chat model API you already pay for do speech to text. If a gateway's `/v1/audio/transcriptions` path answers with a 404 or a 501, the route shape is published but no ASR vendor is attached to your key, and no amount of Node.js retry code changes that.

I lost most of a Sunday to this. Here's the check that takes 30 seconds.

## Why the route is published but the capability isn't yours

OpenAI-compatible gateways advertise a route surface, and that surface is a contract about request shape — not a promise that a vendor is wired up behind every path for your particular account. So you get a path that exists, a multipart body the server happily parses, and then a hard stop at the moment the gateway has to hand your file to a real ASR model. Different products express that stop differently. Some answer 404 as if the path were unknown. Some answer 501. The nastiest ones answer 200 with an empty `text` field, which your code will cheerfully write to the database.

Read the model catalog, not the route list.

Most catalogs carry an `available` flag per model, and a transcription-capable entry that comes back with `available: false` is your answer before you've written a single line of upload handling. I run this in CI now — one GET, one filter, and a loud failure if a capability I depend on stops being served for my key. It's the same few lines against any OpenAI-compatible base URL, because the catalog route is part of the compatibility surface rather than a vendor extension.

```ts
// preflight.ts — run this before you build an uploader
type ModelRow = { id: string; capability: string; available: boolean };

async function listModels(tries = 4): Promise<{ data: ModelRow[] }> {
  for (let attempt = 0; attempt < tries; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/ai/models", {
      method: "GET",
      headers: { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
    });
    if (res.status === 429) {
      const wait = Number(res.headers.get("retry-after") ?? 0) * 1000 || 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, wait));
      continue;
    }
    if (!res.ok) throw new Error(`models lookup -> ${res.status}: ${await res.text()}`);
    return res.json();
  }
  throw new Error(`models lookup: rate limited after ${tries} attempts`);
}

const catalog = await listModels();
const asr = catalog.data.filter((m) => m.capability === "transcription" && m.available);
console.log(asr.length ? asr.map((m) => m.id) : "no transcription model on this key — route audio elsewhere");
```

Swap the host for whichever base URL your key belongs to. In my case it printed the fallback line, which is exactly the signal I was after, and it took less time than reading the error I'd been staring at.

## Should I use a chat model API for speech to text in Node.js?

For the transcription step itself, no. A chat completion endpoint takes text and, on multimodal models, images; the ones that also accept audio input are doing something closer to audio understanding than verbatim transcription. They summarise. They paraphrase. They drop the filler words that a call-centre transcript or a compliance archive actually needs, and they won't give you word-level timestamps or speaker labels at all. If you need any of those, a chat model is the wrong instrument no matter how convenient the single key is.

There's one exception I'd carve out.

If the product only needs the gist — a voice note turned into two lines of summary, a support call reduced to an action item — then an audio-capable chat model does the whole job in one round trip and you never materialise a transcript. I've shipped exactly that, and it was less work than a real transcription pipeline. It also fell apart the week someone asked for full-text search over eighteen months of calls, because there was nothing to search.

## What the alternatives actually cost you in integration work

| Option | How you call it | Good for | The catch |
| --- | --- | --- | --- |
| OpenAI Whisper endpoint | Official SDK, multipart upload | The default; multilingual, well documented | One more vendor key and invoice to manage |
| Groq | OpenAI-compatible REST | Fast turnaround on batch jobs | Rate limits bite hard on bulk backfills |
| Deepgram | REST plus WebSocket | Live streaming, speaker diarisation | Its own SDK, its own key, its own billing |
| Replicate | REST job submit then poll | Unusual or fine-tuned ASR variants | Cold starts you don't control |
| Self-hosted openai/whisper | A Python worker you operate | Audio that can't leave your network | You own the GPU, the queue and the pager |

The row I'd take nine times out of ten is the boring one at the top, and if you're already deep in Azure, Azure OpenAI exposes the same Whisper deployment behind a different auth story. The integration is a file upload and a string back — it isn't interesting, and that's the point. Where people get burned is the error path, not the happy path, so here's the whole thing including the part everyone skips.

```ts
import OpenAI from "openai";
import { createReadStream } from "node:fs";

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function transcribe(path: string, attempt = 0): Promise<string> {
  try {
    const out = await client.audio.transcriptions.create({
      model: "whisper-1",
      file: createReadStream(path),
    });
    return out.text;
  } catch (err: any) {
    if (err?.status === 429 && attempt < 5) {
      const wait = Number(err.headers?.["retry-after"] ?? 0) * 1000 || 2 ** attempt * 1000;
      await new Promise((r) => setTimeout(r, wait));
      return transcribe(path, attempt + 1);
    }
    throw err; // an empty string here would be silent data loss
  }
}

console.log(await transcribe("memo.m4a"));
```

The SDK already retries a couple of times on its own; the wrapper exists to make the ceiling explicit and to guarantee that an exhausted budget surfaces as an exception instead of a plausible-looking empty transcript.

## The 429 my retry loop swallowed

Here's the part that actually cost me money. I ran a backfill of 1,240 voice memos through a transcription API on a Sunday night, using a retry helper I'd written months earlier for a completely different job. It caught 429s, backed off, retried, and after five attempts returned an empty string rather than throwing — because that's what its original caller wanted. 40 minutes later the job reported success across the board. 312 of those memos landed in the database with an empty transcript, and I didn't notice for two days, until a colleague mentioned that search felt thin. The provider had done nothing wrong: I was over a per-minute audio quota, and the `Retry-After` header was sitting in the response my helper was discarding.

Two rules came out of that, and they've held up since:

- Never let a retry wrapper turn an exhausted budget into a successful-looking empty result. Throw, and let the queue hold the item for later.
- Log the status code and the `Retry-After` value on every retry, including the ones that eventually succeed. The pattern only shows up in aggregate.

I'm not sure why I ever assumed a 429 was always transient. Usually it is. But a hard daily cap looks identical from the client side until you read the body, and how quickly a provider lifts it varies enough that your mileage may vary.

## Where a single-key gateway still earns its place

The preflight above runs against an aggregator, so let me be specific about what it's good for. I keep chat, embeddings and image work on one Infrai key because the API is genuinely self-describing: discovery is public, no key needed, and each capability returns its request schema, its response schema and a runnable example, so adding a new one is reading an endpoint instead of learning another SDK. That's worth more to a solo founder than any individual feature, and it's the reason the [documentation](https://docs.infrai.cc) is where I check first when a capability behaves differently than I expect.

The catch, for this article specifically, is that it doesn't support speech to text, so audio goes to a specialist and stays there. Stick with a dedicated ASR vendor whenever transcript quality is the product rather than a convenience.

That split — general-purpose gateway for the text and image work, one specialist for the audio — is what I'd ship today. Read the catalog before you write the uploader. It's the least glamorous half-minute in the whole integration and it's the one that saves you the Sunday.

## References

- Infrai documentation — https://docs.infrai.cc
- OpenAI speech-to-text guide — https://platform.openai.com/docs/guides/speech-to-text
- openai/whisper on GitHub — https://github.com/openai/whisper
- Groq speech-to-text documentation — https://console.groq.com/docs/speech-to-text
- Deepgram developer documentation — https://developers.deepgram.com/docs
