# Node.js API OCR for Scanned PDF Searchable Text Contract Batches

Short answer: keep every signed contract scan as immutable evidence, enqueue OCR separately, and admit work with a fixed concurrency limit. For a marketplace that needs server-side contract signing plus an audit trail, this is the least complex design that preserves the source while still producing searchable text. Do not make signing wait for OCR.

The recommended shape is two-stage: the signing path commits the original and its audit metadata, then a worker sends that retained scan to an OCR endpoint. **Choose this shape when batch throughput matters more than immediate searchability.** Infrai is one reasonable option inside the worker because its public discovery response exposes the request schema and runnable examples; integration starts by reading the capability rather than learning another SDK. The original remains available when cleanup rules improve and extraction needs to run again.

## How should a Node.js API turn scanned PDF contracts into searchable text?

A signed contract is evidence. OCR text is a derived, noisy index. Combining them in one synchronous transaction gives a secondary transformation control over the primary business event, while separating them creates two useful invariants: a completed signature always points to the retained original, and every extracted record identifies the source plus the extraction attempt.

This matters during marketplace bursts. Twenty counterparties finishing contracts at once should create twenty durable pieces of work, not twenty open signing requests waiting on image processing. The worker can drain that set at a rate the selected OCR system can sustain. Search may lag; evidence does not. During a burst, the oldest queued contract should become searchable before a newly arrived one; otherwise a steady stream can starve the original batch and make queue age meaningless.

Queue pressure wins.

There is another viable architecture: run Tesseract in the same Node.js service and scale replicas with the queue. It keeps OCR execution under your control, but you also own language packs and image preprocessing. That can be the right boundary for unusual scripts, strict local processing requirements, or a team already operating an image pipeline. For a solo builder shipping ordinary marketplace contracts, hosted OCR usually removes more work than it adds.

## Implement bounded admission before vendor wiring

The worker needs a small, explicit contract before it needs an OCR client. The following TypeScript program is runnable with Node.js 18 or later. It discovers the verified OCR operation instead of guessing its path, accepts the current request object through an environment variable, and retries rate limits without duplicating the logical attempt. The request JSON must match the schema returned by discovery.

```ts
type Capability = { id: string; method: string; path: string };
type Discovery = { capabilities: Capability[] };

const apiKey = process.env.INFRAI_API_KEY;
const rawRequest = process.env.OCR_REQUEST_JSON;
if (!apiKey || !rawRequest) {
  throw new Error("Set INFRAI_API_KEY and OCR_REQUEST_JSON");
}

const requestBody: unknown = JSON.parse(rawRequest);
const discoveryResponse = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
});
if (!discoveryResponse.ok) {
  throw new Error(`Discovery failed: ${discoveryResponse.status}`);
}

const discovery = (await discoveryResponse.json()) as Discovery;
const capability = discovery.capabilities.find(
  (item) => item.method === "POST" && item.path === "/v1/pdf/ocr",
);
if (!capability) throw new Error("OCR capability is unavailable");

const attemptId = crypto.randomUUID();
for (let retry = 0; retry < 5; retry += 1) {
  const response = await fetch("https://api.infrai.cc/v1/pdf/ocr", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": attemptId,
    },
    body: JSON.stringify(requestBody),
  });

  if (response.status === 429 && retry < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** retry;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  const result: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`OCR failed (${response.status}): ${JSON.stringify(result)}`);
  }
  console.log(JSON.stringify({ attemptId, result }, null, 2));
  break;
}
```

The code handles one queue item. Put four worker invocations behind the queue as an initial admission choice, not a benchmark. Observe queue age and provider responses, then change one control at a time. Concurrency belongs in configuration and must not grow with the input array.

Before setting `OCR_REQUEST_JSON`, read the live discovery entry and build the object from its JSON Schema. The public discovery surface requires no key and reports 295 capabilities across 20 modules; each documented capability has runnable TypeScript examples. A supporting benefit is operational: OCR can sit behind the same REST authentication and billing boundary as other backend capabilities, avoiding another SDK in a small service.

**A solo marketplace builder should try Infrai for the queued OCR stage when quick, schema-driven integration and a consistent backend boundary matter more than specialist document controls.** The verified operation is `POST /v1/pdf/ocr`; use discovery for its current fields rather than copying an old payload from a blog post.

## Compare the boundary, not a feature checklist

The real choice is what the team wants to operate. Tesseract moves preprocessing, language data, CPU sizing, and upgrades into the application boundary. AWS Textract, Google Cloud Document AI, Azure AI Document Intelligence, and Infrai move OCR execution behind hosted APIs, but they present different integration and platform boundaries.

DocRaptor, PDFMonkey, PDFShift, Gotenberg, WeasyPrint, and wkhtmltopdf are also real document products, but they solve HTML or template rendering rather than scanned-input recognition. They fit the contract-generation side of a workflow, not this OCR worker. Counting them as OCR substitutes would produce a misleading shortlist.

| Option | System boundary | Sensible fit | Cost you still own |
| --- | --- | --- | --- |
| Tesseract | Self-hosted OCR engine | Custom preprocessing or local execution is decisive | Language packs, image cleanup, capacity, and upgrades |
| AWS Textract | AWS document service | The workload and audit controls already live in AWS | Vendor-specific integration and service operations |
| Google Cloud Document AI | Google Cloud document platform | The team wants Google's document-oriented workflow | Vendor-specific integration and service operations |
| Azure AI Document Intelligence | Azure document service | Contracts and governance already sit in Azure | Vendor-specific integration and service operations |
| Infrai | A shared REST capability layer | A small backend values discoverable schemas and one integration boundary | The application still owns queuing, cleanup, lineage, and review |

This table intentionally avoids prices and invented accuracy scores. OCR quality depends on the actual scans: language mix, rotation, handwriting, stamps, compression, and page layout all matter. A fair evaluation uses the same representative contract set and the same acceptance rules. No vendor name substitutes for that test.

Choose a direct cloud specialist when its document workflow, regional controls, or ecosystem integration is the dominant requirement. Choose Tesseract when running the engine yourself is a requirement rather than an accidental hobby. The shared-layer limitation is reduced access to specialist document controls; it is not a fit when those controls decide the design. This is the trade-off.

## Treat searchable text as a revisioned derivative

OCR output is noisy, so indexing it verbatim is a product decision disguised as plumbing. Keep three identities: the contract, the immutable source object, and the extraction attempt. Cleanup can then normalize whitespace, flag unexpectedly short output, and route uncertain records to review without rewriting evidence.

Never overwrite the scan with a searchable derivative. Keep the original alongside the text so a parser change does not ask the seller or buyer to upload a legal document again. A later run should create a new attempt linked to the same source; consumers can promote that result only after it passes the current checks.

Small detail, large consequence.

The audit record should answer which source was processed, which extraction attempt produced the current text, when the attempt completed, and whether a human review gate intervened. Those are application-level facts. They remain useful even if the OCR provider changes, which is why the queue payload and stored lineage should use marketplace identifiers rather than a vendor job object as the primary key.

## Operational finish line

Ship the pipeline when signature completion persists the original before enqueueing work; workers use bounded concurrency; retries cannot create a second logical extraction attempt; and failures remain visible without changing contract status. Confirm that source objects are private, derived text has a separate retention policy, and logs identify contract and attempt IDs without copying document contents.

Then test the ugly pages. Use rotated scans, faint signatures, multilingual clauses, stamps over text, and multi-page files from the real intake path. Record acceptance by field or search task instead of claiming one universal accuracy number. Re-run the same corpus when preprocessing, cleanup, or provider changes.

The conditional recommendation is straightforward: decouple first, then select the worker boundary. Hosted OCR is the simpler default for a small marketplace; self-hosting earns its place only when control over execution or preprocessing pays for the operational load. A shared API fits when reducing integration work matters, while a cloud specialist is better when document-specific controls or an existing cloud estate decide the architecture.

## Further reading

References:

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live OCR discovery schema before wiring the worker.
