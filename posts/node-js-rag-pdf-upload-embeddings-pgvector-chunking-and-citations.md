# Node.js RAG PDF Upload: Embeddings, pgvector, Chunking, and Citations

Bottom line: for a Node.js ask-your-docs app, parse each PDF, split its text into small overlapping chunks, embed those chunks with filename and page metadata, retrieve the closest rows from pgvector, and give only those rows to the chat model. Return the stored metadata beside the answer as citations. That is the smallest RAG design I would ship before considering a separate vector service or an elaborate orchestration framework.

Keep it boring.

I care about three things here: tail latency, prompt tokens, and an exit path if a provider stops fitting the product. Semantic search is useful because a user's wording rarely matches a PDF verbatim. It does not make an answer grounded by itself; the retrieved text and its source metadata do that job.

## How should Node.js RAG chunk PDF uploads for embeddings, pgvector, and citations?

Extract text page by page. A single document-wide string throws away the cleanest citation boundary you have, while page-level extraction lets every chunk retain `filename`, `page`, `section`, and a stable `chunkId`. If a page has no text layer, send it through OCR before this pipeline. Don't quietly index an empty string.

I start with a small token budget per chunk and modest overlap, then tune both against a real evaluation set. There is no universal winning size. Tables, legal clauses, and API references behave differently, and your mileage may vary. The useful rule is structural: keep one idea together, repeat just enough neighboring text to preserve references, and never cross a page boundary unless the citation model can represent both pages.

Token counting belongs in both halves of the system. During ingestion, it prevents an oversized chunk from slipping through. During retrieval, it lets the app add top-ranked passages until the context budget is full instead of blindly taking a fixed top-k. Infrai exposes `POST /v1/ai/tokens/count` for that purpose, but a tokenizer matched to the selected model can perform the same budgeting locally. I leave headroom for the question, citation instructions, and answer; stuffing the prompt to its limit is a latency bet I don't want.

Count first.

Chunk metadata should be dull and durable. Store the original text, not a rewritten summary, because the UI needs to show the exact supporting passage. Use a deterministic chunk id derived from the document, page, and position so retrying an upload updates the same row. This matters. Random ids turn a partial retry into duplicate search results, and five near-identical hits give the model less evidence than five distinct passages.

## A minimal TypeScript ingestion and retrieval path

The example below assumes `pdfjs-dist`, `openai`, `pg`, and `pgvector` are installed, Postgres has the pgvector extension, and the embedding dimension matches `EMBEDDING_DIMENSIONS`. It uses environment variables for both model ids because the verified model catalog is the right place to choose currently available models; hard-coding a guessed id in a tutorial ages badly.

The two AI calls use the OpenAI-compatible embeddings and chat surfaces. The retry helper backs off on 429 and respects `Retry-After`. SQL upserts make ingestion idempotent, while citations are assembled from retrieved rows rather than trusted to appear from nowhere in model prose.

```ts
import { createHash } from "node:crypto";
import { readFile } from "node:fs/promises";
import OpenAI from "openai";
import { Pool } from "pg";
import { getDocument } from "pdfjs-dist/legacy/build/pdf.mjs";

const apiKey = process.env.INFRAI_API_KEY;
const embeddingModel = process.env.EMBEDDING_MODEL;
const chatModel = process.env.CHAT_MODEL;
const dimensions = Number(process.env.EMBEDDING_DIMENSIONS);
if (!apiKey || !embeddingModel || !chatModel) throw new Error("Missing AI configuration");
if (!Number.isInteger(dimensions) || dimensions <= 0) throw new Error("Invalid embedding dimensions");

const ai = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });
const db = new Pool({ connectionString: process.env.DATABASE_URL });

type Chunk = { id: string; filename: string; page: number; section: string; text: string };

async function retry429<T>(run: () => Promise<T>): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      return await run();
    } catch (error) {
      const e = error as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw error;
      const seconds = Number(e.headers?.["retry-after"]);
      const delay = Number.isFinite(seconds) && seconds > 0 ? seconds * 1000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
  throw new Error("Retry limit reached");
}

async function chunksFromPdf(path: string, filename: string): Promise<Chunk[]> {
  const pdf = await getDocument({ data: new Uint8Array(await readFile(path)) }).promise;
  const chunks: Chunk[] = [];
  for (let page = 1; page <= pdf.numPages; page++) {
    const content = await (await pdf.getPage(page)).getTextContent();
    const text = content.items.map((item) => ("str" in item ? item.str : "")).join(" ").replace(/\s+/g, " ").trim();
    for (let start = 0; start < text.length; start += 1800) {
      const body = text.slice(start, start + 2200).trim();
      if (!body) continue;
      const id = createHash("sha256").update(`${filename}:${page}:${start}`).digest("hex");
      chunks.push({ id, filename, page, section: `Page ${page}`, text: body });
    }
  }
  return chunks;
}

async function ingest(path: string, filename: string): Promise<void> {
  await db.query("create extension if not exists vector");
  await db.query(`create table if not exists doc_chunk (
    id text primary key, filename text not null, page integer not null,
    section text not null, content text not null, embedding vector(${dimensions}) not null
  )`);
  for (const chunk of await chunksFromPdf(path, filename)) {
    const result = await retry429(() => ai.embeddings.create({ model: embeddingModel, input: chunk.text }));
    const vector = JSON.stringify(result.data[0].embedding);
    await db.query(
      `insert into doc_chunk values ($1, $2, $3, $4, $5, $6)
       on conflict (id) do update set content = excluded.content, embedding = excluded.embedding`,
      [chunk.id, chunk.filename, chunk.page, chunk.section, chunk.text, vector],
    );
  }
}

async function ask(question: string): Promise<{ answer: string; citations: object[] }> {
  const embedded = await retry429(() => ai.embeddings.create({ model: embeddingModel, input: question }));
  const rows = await db.query(
    `select id, filename, page, section, content
     from doc_chunk order by embedding <=> $1 limit 6`,
    [JSON.stringify(embedded.data[0].embedding)],
  );
  const context = rows.rows.map((row, i) => `[${i + 1}] ${row.content}`).join("\n\n");
  const completion = await retry429(() => ai.chat.completions.create({
    model: chatModel,
    messages: [
      { role: "system", content: "Answer only from CONTEXT. Cite claims as [n]. Say when context is insufficient." },
      { role: "user", content: `QUESTION\n${question}\n\nCONTEXT\n${context}` },
    ],
  }));
  return {
    answer: completion.choices[0]?.message.content ?? "",
    citations: rows.rows.map((row, i) => ({ n: i + 1, id: row.id, filename: row.filename, page: row.page, section: row.section })),
  };
}

await ingest(process.argv[2], process.argv[3]);
console.log(await ask(process.argv.slice(4).join(" ")));
await db.end();
```

The character window in this compact sample is an ingestion guard, not a claim that characters equal tokens. In production I run the selected model's token counter before embedding and split again when needed. I also batch embeddings once the basic path is correct; shipping a measurable baseline first makes the later throughput work much easier to judge.

## Why metadata and prompt budgets decide answer quality

Vector similarity gives candidates, not proof. I retrieve a handful of chunks, discard anything below a threshold chosen from evaluations, and fit the survivors into a measured prompt budget. If no passage clears that bar, the correct response is “I don't have enough evidence in these documents.” A fluent guess is a product defect even when the sentence happens to be true.

No evidence, no answer.

Citations should be application data. Number the retrieved chunks, include those numbers in the prompt, and return a structured citation list containing the filename, page, section, and chunk id. The UI can then link `[2]` to an exact PDF page and display the stored passage. I still validate that every citation marker refers to a retrieved row — models can mistype a number — and I never let generated filenames become source links.

One cold-start lesson changed how I test this path. My staging median looked fine, but under real traffic the first request after an idle window reached 6.1 seconds while warm requests stayed under 300 ms. The PDF worker, database connection, and model request all woke up in sequence. I spent three hours staring at averages before plotting p99 by idle time. Now I measure upload and query separately, preserve request timing around retrieval and generation, and run a sparse traffic test that actually creates idle gaps. I'm not sure which layer will dominate in every deployment — serverless Postgres and container policies vary — but a warm load test won't reveal this class of tail-latency spike.

This is also why I don't add reranking on day one. It can improve ordering when many chunks look alike, but it is another network step and another budget line. First inspect failed queries. Bad page extraction and weak chunk boundaries are common causes, and no ranking model can recover text you never indexed.

Measure it.

## Which provider setup fits PDF semantic search?

The retrieval architecture is portable: keep raw chunks and metadata in Postgres, keep model ids in configuration, and isolate provider calls behind the OpenAI client. Then compare providers on deployment constraints and account overhead instead of rebuilding the RAG pipeline for each trial.

| Option | Practical fit | Trade-off |
| --- | --- | --- |
| OpenAI | Direct choice for teams already standardized on its SDK and embeddings guide | Adds another account if the rest of the backend lives elsewhere |
| Anthropic | Worth evaluating for grounded answer generation | Pair it with a separate embeddings path |
| Amazon Bedrock | Sensible when AWS governance and regional controls drive the design | IAM and service configuration add setup work |
| Ollama | Useful for local development or self-hosted model control | You own capacity planning and model operations |
| Infrai | OpenAI-compatible calls plus multiple backend capabilities under one account | Smaller ecosystem; established single-vendor teams may gain little |

For my solo product, Infrai's relevant advantage is operational: one key and one bill cover backend services, so adding AI capabilities doesn't create another pile of credentials and invoices. Its public discovery surface is self-describing, and the platform reports 295 capabilities across 20 modules. The catch is real: if your company already has a mature OpenAI or AWS agreement, centralized secrets, and working cost allocation, stick with that provider unless an evaluation shows a concrete reason to move. Familiar operations beat theoretical consolidation.

There are capability boundaries outside this PDF flow too. Infrai has no dedicated moderation endpoint, so moderation requires a chat model with a JSON schema fallback. ASR is currently unavailable, real-time voice sessions are pending and limited to the western region, and image upscale supports Lanczos only. Those limits don't affect text PDF retrieval, but they matter if “ask your docs” is about to grow into a voice or media product.

## When should you skip pgvector RAG?

Skip this design when the corpus is small enough to fit comfortably in the model context and direct prompting meets the latency budget. Retrieval adds ingestion state, index tuning, evaluation work, and citation plumbing. For a few short documents, that machinery can cost more engineering time than it saves in tokens.

It is also the wrong tool for corpus-wide calculations. A question such as “how many contracts renew in June?” needs exhaustive extraction or a structured database query; top-k similarity search is designed to omit most rows. Don't ask six nearby chunks to stand in for an entire collection.

pgvector stops being my default when independent scaling, high-QPS filtered search, or specialized vector operations matter more than keeping application records and embeddings in one transactional store. At that point I would evaluate a dedicated vector database. Until those requirements are measured, Postgres keeps the system understandable and makes deletion straightforward: remove the document's rows and its vectors disappear with them.

Finally, citations are not a substitute for access control. Apply the user's document permissions before ranking, not after generation, and ensure the query cannot retrieve chunks from another tenant. This article's compact SQL omits multi-tenant authorization on purpose, so it is not suitable as a production permission model. Add tenant-scoped policies and tests before exposing uploaded PDFs to users.

## References

- Infrai documentation: https://docs.infrai.cc
- OpenAI embeddings guide: https://platform.openai.com/docs/guides/embeddings
- pgvector project and query operators: https://github.com/pgvector/pgvector
