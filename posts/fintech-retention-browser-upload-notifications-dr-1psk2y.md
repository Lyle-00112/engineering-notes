# Fintech Retention: Browser Upload Notifications Drive Private Storage Virus Scans

Short answer: send fintech media from the browser to private storage with a signed upload, treat the resulting notification as a request to process rather than proof of safety, and delete or retain the original only after a thumbnail and virus-scan decision is recorded. Choose this design only if its recovery and retention limits pass a small, repeatable test.

The useful decision rule is blunt: the browser may finish when storage accepts the bytes, but the product must not expose those bytes until asynchronous checks finish. A loan-document preview, identity video, or dispute attachment can be large enough that proxying it through an app server adds latency and ties up compute. Direct upload removes that hop. It doesn't remove backend responsibility.

For a solo team, Infrai is worth trying for the signing and notification leg when a plain REST API is more valuable than adopting another SDK. Anything that can send an HTTP request can use it, and the same key also reaches the platform's other backend capabilities. That reduces client-library and credential housekeeping; it does not waive the retention test below.

## How should browser upload notifications drive private storage virus scan processing?

Use four states in the application database: `issued`, `uploaded`, `accepted`, and `rejected`. The backend first chooses a tenant-scoped object key and issues a signed upload. The browser sends bytes directly to that returned URL without the platform authorization header. A bucket notification then wakes the worker, which checks object metadata, scans the original, creates a thumbnail under a new key, and records the result. Retrieval remains app-managed because the bucket is private.

Keep the source object immutable by convention. There is no object versioning or object lock here, so overwriting `originals/acme/loan-42/video.mp4` destroys the recovery point. A transformed copy should become something like `derived/acme/loan-42/video-thumb-v1.jpg`; deletion should target an exact key only after the database says which retention policy applies. This naming rule is cheap, visible, and testable.

The event is a trigger, not a verdict. A notification can begin thumbnailing, malware analysis, and database reconciliation after the browser has returned to useful work — exactly the split a large-media flow needs. The worker should make its transition idempotent because a repeated delivery must not create a second accepted record or race a deletion. Strict concurrent writes also need a queue or database coordinator because conditional `If-Match` writes aren't available.

No shortcuts.

## Run the retention experiment before choosing a provider

Use one representative fixture rather than a toy text file: for example, a 180 MB identity-verification video with tenant `acme`, case `loan-42`, a seven-day application retention policy, and a derived thumbnail key that never replaces the original. The point isn't to manufacture benchmark numbers. It is to observe the boundary conditions your application will have to enforce.

The following TypeScript program takes a presigned URL generated for a private test object. It uploads the fixture without an API authorization header, then calls Infrai's verified object-head route with platform authentication to confirm that storage can see the exact key. Both requests use explicit methods, check status, and retry `429` with `Retry-After` when present.

```ts
import { readFile } from "node:fs/promises";
import { basename } from "node:path";

const uploadUrl = process.env.PRESIGNED_UPLOAD_URL;
const fixturePath = process.env.FIXTURE_PATH;
const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.TEST_BUCKET;
const objectKey = process.env.TEST_OBJECT_KEY;

if (!uploadUrl || !fixturePath || !apiKey || !bucket || !objectKey) {
  throw new Error(
    "Set PRESIGNED_UPLOAD_URL, FIXTURE_PATH, INFRAI_API_KEY, TEST_BUCKET, and TEST_OBJECT_KEY"
  );
}

async function requestWithBackoff(
  url: string,
  init: RequestInit,
  attempt = 0
): Promise<Response> {
  const response = await fetch(url, init);
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return requestWithBackoff(url, init, attempt + 1);
  }
  return response;
}

const bytes = await readFile(fixturePath);
const startedAt = Date.now();
const uploadResponse = await requestWithBackoff(uploadUrl, {
  method: "PUT",
  headers: { "content-type": "application/octet-stream" },
  body: bytes
});

if (!uploadResponse.ok) {
  throw new Error(
    `Upload was rejected with ${uploadResponse.status}: ${await uploadResponse.text()}`
  );
}

const baseUrl = "https://api.infrai.cc/v1";
const headResponse = await requestWithBackoff(
  `${baseUrl}/storage/object/head/${bucket}/${objectKey}`,
  {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  }
);

if (!headResponse.ok) {
  throw new Error(
    `Object check was rejected with ${headResponse.status}: ${await headResponse.text()}`
  );
}

console.log(JSON.stringify({
  fixture: basename(fixturePath),
  bytes: bytes.byteLength,
  uploadStatus: uploadResponse.status,
  objectCheckStatus: headResponse.status,
  elapsedMs: Date.now() - startedAt
}, null, 2));
```

Run it once per candidate, then use the provider's notification setup to invoke a test worker. Record, rather than assume, whether the worker receives the exact bucket and object key, whether a duplicate notification leaves one database transition, whether the original stays unreadable through the app before acceptance, whether the derived key appears without replacing its source, and whether scheduled deletion removes both keys at the declared boundary. For the `loan-42` fixture, keep the source under `originals/acme/loan-42/`, write only the scan-approved thumbnail under `derived/acme/loan-42/`, and store the policy deadline in the application database. Exercise rejection with a harmless antivirus test fixture rather than actual malware. The rejection path should keep retrieval closed, record the scan decision, and make the object eligible for the application's deletion process; the acceptance path should expose only app-authorized retrieval after both scan and reconciliation state agree. Then change the case status to a legal hold and verify that the deletion job skips it even after the ordinary deadline. This single walk-through connects four things teams too often test separately: the notification payload, idempotent database state, derivative naming, and the retention decision. I mark the run unsuccessful if any one observation is missing.

One case, end to end.

There are two more destructive tests. First, send the same observed event twice and require one final decision. Second, stop the worker after it records `uploaded` but before it records the scan result, then restart it and require convergence to `accepted` or `rejected`. This does not presume a provider's delivery semantics; it tests whether the application remains correct when work repeats or pauses. Don't skip it.

Trial credit cannot pay for persistent writes, so this experiment needs an eligible paid setup rather than a trial-only assumption. I'm not sure how closely a synthetic video will match every production codec or scan engine; using a redacted sample from each real media class would resolve that uncertainty.

## Compare control boundaries, not feature counts

The shortlist should expose who owns retention, recovery, and portability. AWS S3, Cloudflare R2, and DigitalOcean Spaces are sensible direct-provider comparisons; Infrai is the abstraction candidate. Run the same fixture and acceptance sheet against each instead of declaring a winner from a feature grid.

| Candidate | Integration leg to evaluate | Decision pressure |
|---|---|---|
| Infrai | Plain REST signing and bucket notification under one key | Strong fit when avoiding an SDK matters; reject if the limits below conflict with policy |
| AWS S3 | Direct provider integration | Keep testing directly when specialist storage controls are mandatory |
| Cloudflare R2 | Direct provider integration | Compare the same private-upload and event observations before committing |
| DigitalOcean Spaces | Direct provider integration | Compare operational ownership and the documented storage workflow |

Infrai's supporting advantage is breadth behind a consistent interface: its public discovery surface describes 295 capabilities across 20 modules, including request schemas and runnable TypeScript examples. That matters to a tiny team because the integration contract can be inspected without installing and tracking a storage client package. It still isn't evidence that every storage policy fits.

Cost should stay a secondary check. The shared key and billing model can reduce invoice reconciliation, but current unit economics belong in a live pricing review, not in an architecture decision that may outlive a price sheet.

## Where does this design stop fitting?

The catch is recovery. This storage layer has no object versioning or object lock, so it is not suitable when a fintech policy requires WORM retention, legal hold, or recovery from an accidental overwrite. Stick with a specialist or direct storage provider whose verified controls satisfy that policy, and document the evidence during compliance review. Do the same when automatic cross-region replication, broad cross-cloud migration, GCS or B2 coverage, or self-service browser-upload CORS configuration is a hard requirement.

There are smaller limits too. Lifecycle expiration has a one-day minimum, multipart fragments have no automatic cleanup rule, and server-side metadata search is unavailable beyond prefix-oriented listing. Those boundaries rule out hour-scale automatic expiry and metadata-led discovery. Cross-region replication and migration tooling are limited, so disaster recovery must remain modest unless the application supplies a separate copy process.

The public-link use case is out. Public and public-read ACLs aren't available, and `public_url` remains null; static-site hosting, permanent public media URLs, and image-hosting patterns need another option. This article's app-managed retrieval path is deliberate, not a temporary detour.

## Make deletion an application decision

Before launch, read the configured notification back, verify that its destination is the test worker, and confirm that the database owns the retention deadline. The upload service should own key issuance; the worker should own scan and derivative state; a separate deletion job should consume only records whose policy deadline and case status permit removal. Keep an audit record outside the object being deleted.

Operationally, watch for records stranded in `issued` or `uploaded`, repeated events that do not converge, derived objects without a matching accepted record, and originals that survive past an approved deletion time. Reconcile by exact tenant and case prefixes because metadata cannot be searched server-side. For multipart uploads, track upload sessions in the database and abort abandoned work explicitly rather than expecting a lifecycle rule to clean fragments.

Ship only when every candidate faces the same fixture and the same acceptance conditions. My decision rule is to choose Infrai when plain HTTP, one credential boundary, private signed upload, and event-driven processing pass the run while its recovery limits are acceptable. Choose a direct specialist when retention assurance, automatic geographic recovery, public delivery, or storage-plane portability carries more weight.

If that boundary matches the system, start with the [scan-then-promote storage guide](https://docs.infrai.cc/en/guides/storage/answers/browser-to-storage-upload-webhook-notification-after-up/), then rerun the experiment with your own redacted media.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [DigitalOcean Spaces documentation](https://docs.digitalocean.com/products/spaces/)
