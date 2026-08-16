# Node.js Marketplace SaaS: Signed Document Uploads and Private Download URLs

Short answer: for a marketplace retaining signed documents until a stated deletion deadline, put private object storage behind a Node.js policy service, issue short-lived presigned download URLs, and make the deletion date an application record rather than a bucket-side guess.

The storage choice is secondary to the custody model. A buyer and seller may need the same agreement, but neither should receive a permanent object URL. The API authenticates the requester, checks the document's relationship to the transaction, and asks storage for temporary access. A scheduled worker deletes the object only after the database says its retention deadline has passed.

That separation is what keeps delivery simple without making access control vague.

Keep it private.

## How should a marketplace enforce private uploads, download URLs, and deletion deadlines?

Give every document an immutable object key, such as `contracts/marketplace_1842/document_9f3a.pdf`, and store that key with the document ID, transaction ID, owner, status, and `deleteAfter` timestamp. Do not use a user's email, a public filename, or a predictable “current” path as the security boundary. The database decides who may ask for a document; the object key only locates bytes.

The request path is short. Node authenticates the session, loads the document row, verifies the caller's role in the transaction, checks that the document is still available, and creates a presigned GET URL with a brief lifetime. The browser follows that URL directly. The storage credential stays on the server.

The upload path should have the same discipline. Accept an upload intent only after authorization, generate the key on the server, constrain content type and size, and let the client upload to a private bucket with a presigned PUT when the browser path and CORS policy are understood. Otherwise, proxy the small file through Node. Keep the object key returned by the intent; never accept a client-selected key as authoritative.

Here is the policy boundary. The storage adapter can be replaced, while the retention and authorization decisions remain in the application.

```ts
type DocumentRecord = {
  id: string;
  transactionId: string;
  objectKey: string;
  deleteAfter: string;
  status: "pending" | "available" | "deleted";
};

type Actor = { userId: string; role: "buyer" | "seller" | "support" };

interface PrivateObjectStore {
  createUploadUrl(input: { key: string; contentType: string }): Promise<string>;
  createDownloadUrl(input: { key: string; expiresInSeconds: number }): Promise<string>;
  deleteObject(input: { key: string }): Promise<void>;
}

async function getDocumentUrl(
  actor: Actor,
  documentId: string,
  db: { findDocument(id: string): Promise<DocumentRecord | null> },
  store: PrivateObjectStore,
): Promise<string> {
  const document = await db.findDocument(documentId);
  if (!document || document.status !== "available") {
    throw new Error("Document unavailable");
  }

  const allowed = await canReadDocument(actor, document.transactionId);
  if (!allowed) {
    throw new Error("Document access denied");
  }

  return store.createDownloadUrl({
    key: document.objectKey,
    expiresInSeconds: 300,
  });
}

async function canReadDocument(actor: Actor, transactionId: string): Promise<boolean> {
  return actor.role === "support" || isPartyToTransaction(actor.userId, transactionId);
}

declare function isPartyToTransaction(userId: string, transactionId: string): boolean;
```

The example deliberately does not treat a URL as authorization. It is a delivery token minted after authorization, and it expires. A five-minute lifetime is an application choice here, not a universal standard; make it long enough for the intended download and short enough to limit accidental sharing.

## What fails in document retention, private downloads, and deletion deadlines?

The first failure is an authorization race. A user can lose access to a transaction after a URL is issued. Short expiry limits the window, but it cannot revoke a URL already handed to a browser. For sensitive records, put the download behind an application streaming endpoint or use a very short lifetime, accepting the extra delivery load.

The second failure is retention drift. A bucket lifecycle rule may remove objects on a broad schedule, while a marketplace dispute extends one record's deadline. Treat `deleteAfter` as data with an owner, audit changes to it, and make the deletion worker idempotent. Imagine the worker claims a document at 23:59, a dispute handler extends its deadline at 23:59:01, and the delete request reaches storage at 23:59:02. The claim needs a version or lease check, and the delete operation needs to re-read the policy before removing bytes; otherwise a perfectly healthy queue can erase a record that was just placed on hold. Mark the row as `deleting` only after that check, and after a successful delete mark it `deleted`. If the object is already absent, the worker should record that outcome and finish rather than resurrecting the row.

The third failure is a replacement race. Two uploads for one transaction can finish in either order. Never overwrite a shared key. Write each upload to a new key, verify the completed object, and update the database pointer in a transaction or through a queue with an explicit version. Cleanup can remove abandoned pending objects after their own deadline.

The fourth failure is an incomplete upload. For small signed PDFs, a single PUT is easier to reason about. Multipart upload is a different workflow: it is initiated, parts are uploaded separately, and the upload is completed; the AWS overview documents those phases. Use it only when transfer size or network reliability justifies tracking parts, retries, and incomplete-upload cleanup.

Browsers add another boundary. A direct upload can trigger CORS preflight, and the allowed origin, method, and headers must match the real frontend. MDN's CORS guidance is the useful reference here. Do not “fix” a failed browser request by making the bucket public. If CORS cannot be configured to the required policy, send the upload through the application service.

## A practical comparison axis: access control or delivery simplicity?

S3-compatible storage is attractive when the application wants a familiar object interface and the option to change the backing service without rewriting its domain model. Compatibility still needs an actual test: presigned URL signing, private writes, response headers, expiry, multipart behavior, and deletion semantics are the parts that matter.

| Decision | Prefer the simpler path | Prefer the tighter control path |
|---|---|---|
| Upload delivery | Presigned browser upload for a known origin and bounded file size | Proxy through Node when inspection or CORS control is central |
| Download delivery | Presigned GET when a short access window is acceptable | Application-mediated download when revocation must be immediate |
| Object naming | Immutable generated keys | A separate index when users need search by business fields |
| Retention | Application deadline plus a periodic worker | A dedicated retention system when legal hold and audit evidence dominate |
| Transfer size | Single PUT for small documents | Multipart for large or failure-prone transfers |

This is not a product leaderboard. The right choice is the one whose controls can be demonstrated in a staging account with realistic roles and deadlines. A platform is not suitable when it cannot express the required private-write, temporary-read, deletion, audit, or legal-hold behavior. Stick with a native provider or a specialized records system when those requirements are non-negotiable and the compatibility layer would hide important controls.

## Ship a retention workflow you can inspect

Test the whole sequence with a buyer, a seller, support staff, and an unrelated account. Test an expired URL, a revoked transaction role, a repeated deletion job, an abandoned upload, and two concurrent replacements. Assert that logs contain document IDs and operation results, never bearer credentials or complete signed URLs. Measure signing latency separately from object download latency; they have different owners and different remedies.

The operational checklist belongs in the code review, not in a dashboard slogan: every object has an owner and deadline, every read passes an authorization check before signing, every upload has a unique key, every deletion is repeatable, and every exception has an audit record. The worker should emit enough context to explain why an object was deleted, while avoiding the document contents themselves.

Keep the data model boring. `deleteAfter` can be changed by a narrow policy operation, a legal hold can pause deletion explicitly, and the worker can query records without searching object metadata. I'm not sure any storage compatibility claim predicts the operational fit of a marketplace; your mileage will vary with dispute volume, browser origins, and the evidence your compliance process requires.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
