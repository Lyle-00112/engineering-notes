# Node.js Signed URL Downloads for Private Document Storage (Retention Before Egress)

A signed document must become unavailable when its deletion deadline arrives, even if an older download link has not expired. **Short answer:** for private document downloads, put the retention deadline in application state, refuse to create a signed URL after that deadline, delete through a retryable worker, and reconcile the database against object storage before comparing providers on egress cost.

That was the evaluation constraint in my developer-tools test: a build could hold a signed PDF, and the contract required deletion 30 days after the build was deleted. The simple version stored the object, issued a seven-day link, and trusted a nightly cleanup. It was easy to ship. It also made two clocks compete: link expiry and document retention. The design I kept gave the deletion deadline authority over both download authorization and cleanup.

Links linger.

## Preserve retention semantics across a storage migration

A signed URL is a temporary capability to make one storage request. It isn't a retention policy, and shortening its lifetime doesn't prove that a document was deleted. The useful invariant is narrower: once `deleteAfter` has passed, the application must stop minting links; after deletion is confirmed, the object must not appear in reconciliation.

This changes the provider comparison. S3 presigned URLs, R2 signed URLs, Supabase Storage, and Firebase Storage expose different policy and integration surfaces, but the application still needs one answer to a local question: may this document be downloaded now? Keep that answer next to the document record rather than hiding it inside a storage adapter.

There is a catch. A URL that was already issued may remain usable until its own expiry unless a controlled serving layer checks current application state on every request. For documents that require immediate revocation, route downloads through such a check. For documents where bounded link expiry meets the contract, direct signed delivery avoids another transfer hop. I'm not sure which side is right without the actual revocation requirement; sensitivity, file size, latency, and concurrent download volume decide it.

I model three states: `active`, `deleting`, and `deleted`. The record carries an opaque object key and an absolute UTC deadline. URL creation only accepts `active` records before the deadline, while the deletion worker claims due records before touching storage. This is deliberately boring — one policy function can be tested without an SDK, a network, or a particular provider.

```ts
type DocumentRecord = {
  key: string;
  deleteAfter: string;
  state: "active" | "deleting" | "deleted";
};

type ObjectStore = {
  createSignedGet(key: string, expiresInSeconds: number): Promise<string>;
  deleteObject(key: string): Promise<void>;
};

function mayCreateDownload(doc: DocumentRecord, now: Date): boolean {
  return doc.state === "active" && now.getTime() < Date.parse(doc.deleteAfter);
}

async function createDocumentDownload(
  doc: DocumentRecord,
  store: ObjectStore,
  expiresInSeconds: number,
  now = new Date(),
): Promise<string> {
  if (!mayCreateDownload(doc, now)) {
    throw new Error("Document is outside its download window");
  }

  return store.createSignedGet(doc.key, expiresInSeconds);
}
```

The adapter boundary is the portability mechanism. Provider-specific signing belongs behind that boundary, while retention decisions stay unchanged. That doesn't make migration free: authentication, access policy, regional placement, request limits, and observability still differ. It does keep those differences out of the most important rule.

Deletion follows the same separation. A worker claims a due row, changes it to `deleting`, requests object deletion, and marks the row `deleted` only after confirmation. If confirmation is absent, the record remains unavailable for new downloads and eligible for retry. The database is the policy ledger; storage is the resource being reconciled.

Don't erase the ledger entry with the object. A minimal deletion event needs enough information to connect the deadline, object key or stable digest, attempt time, and confirmed completion without retaining the document itself. The exact audit retention period is a legal and product decision, not something a storage price page can settle.

## Put egress math around the authorization path

Cost comes after the deadline test because a cheaper path that cannot demonstrate timely deletion is outside the feasible set. For each candidate, estimate stored byte-months, download egress, request volume, incomplete multipart data, and any transfer introduced by a controlled authorization layer. Use the same traffic trace and retention distribution for every option; otherwise a provider comparison is just mismatched assumptions in a spreadsheet.

Public pricing pages can populate the worksheet, but they can't choose the architecture. Backblaze B2, for example, publishes storage pricing for its own service, while an S3 implementation has separate multipart behavior documented by AWS. Those pages answer different parts of the model. Provider terms and rates change, so capture the region, retrieval path, request mix, and date with each estimate rather than preserving a naked monthly total.

**The limitation is operational weight.** A database ledger, worker, audit event, and reconciler aren't suitable for a prototype that has no private documents or contractual deletion deadline. A storage-native lifecycle rule may be enough when approximate age-based cleanup is acceptable. At the other extreme, stick with native retention or legal-hold controls when records must be preserved against ordinary deletion, and let the compliance system own the deadline. This pattern is for explicit deletion, not immutable archives.

## How can Node.js signed URL downloads survive private document deletion?

The first design depended on the worker being invoked. The stronger design asks a second question on a schedule: which database rows and stored objects disagree? That reversal matters. A queue can report that work was accepted while an object inventory still reveals stale data, and a database can say `deleted` while a later audit finds a key that should no longer exist. Reconciliation catches drift between two systems instead of treating one system's success path as proof about the other.

The focused experiment uses four cases, not a broad benchmark suite:

| Test case | Expected application decision | Evidence to retain |
| --- | --- | --- |
| Active document before its deadline | A short-lived download may be created | Authorization decision and link expiry |
| Active document at or after its deadline | No new download is created | Deadline and denied decision |
| Due document claimed by the worker | State becomes `deleting` before storage access | Claim time and attempt identity |
| Confirmed deletion | State becomes `deleted` and reconciliation finds no object | Confirmation time and inventory result |

## Spend the reliability budget on reconciliation

Multipart uploads need their own line in the inventory model. AWS documents multipart upload as a sequence in which parts are uploaded before a completion request creates the object; its overview also covers stopping an upload and the billing implications of uploaded parts. A retention sweep that only lists completed objects can therefore miss upload state. Track upload identifiers beside provisional document records and include incomplete uploads in cleanup and cost accounting.

This is where I would inject process failures in a test environment: after the row is claimed but before the delete call, and after the delete call but before the row is marked complete. Start with a document due at `2026-08-13T10:00:00Z`, pause the worker at each boundary, and ask the download handler for a new link. It must refuse while the row is `deleting`. Resume the worker, run it again, and then run reconciliation over both completed objects and tracked multipart uploads. The expected result is not a particular provider response. It is that no new link can be minted from `deleting`, retry converges on `deleted`, and reconciliation eventually reports no retained object or incomplete upload for the expired record. Repeat the sequence with two workers claiming the same due set, because exclusivity in a tidy unit test says little about deployment concurrency. Test with realistic PDFs rather than one-kilobyte fixtures because the controlled-download option changes latency and transfer load in ways a tiny object won't expose.

Timing is policy.

One detail is easy to miss: account closure and build deletion should calculate `deleteAfter` once, in a transaction with the state change that starts retention. Recomputing the deadline on every retry lets schedule drift change policy. Keep the original timestamp stable, then measure execution against it.

## Record release evidence for the retention worker

Measure deadline lag: the time from `deleteAfter` to confirmed removal. Also track due records still in `deleting`, reconciliation mismatches, incomplete multipart upload age, denied download attempts after deadline, bytes delivered through each download path, and the ratio of signed links created to completed downloads. The last two expose the real transfer shape without pretending list price is total cost.

Start with shadow reconciliation before letting it alter state. Compare its findings with the worker's audit events, then test account closure and document download together in deployment. A release passes when an expired record cannot mint a link and the reconciler can account for the corresponding storage state. It doesn't pass because the cleanup job logged success.

That is the experiment worth copying: define the deletion invariant, force the awkward timing cases, and price only the architectures that satisfy it. The provider choice can change later. The retention contract shouldn't.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://www.backblaze.com/cloud-storage/pricing
- https://datatracker.ietf.org/doc/html/rfc6750
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/DELETE
