# Tenant-Safe PDF Delivery in Next.js: Regional Object Storage and Signed URLs

Short answer: treat a private PDF export as a state machine with tenant ownership, approved region, and object readiness; only then let a Next.js API route create a short-lived signed download URL. The browser should get one narrow read capability, never storage credentials or a user-selected bucket.

For property management, this is a reliability problem with a security consequence. A tenant-scoped inspection packet or lease summary can be generated correctly and still be delivered incorrectly if a retry, stale database row, or region default selects the wrong object. The implementation has to make an incorrect handoff difficult to represent.

Start with invariants.

## Deployment evidence for the export contract

The test matrix should follow the transitions rather than only checking that a PDF opens. A caller from tenant A requesting tenant B's export must receive no file. A missing region mapping must not fall through to a default. An object that is not ready must not be signed, and an expired URL must be rejected by the storage boundary. Two concurrent requests for one export should converge on the stable destination.

Run both region mappings in CI. Exercise the deployed route with a PDF containing a canary string belonging to exactly one tenant, then inspect the downloaded bytes and the application logs. The logs should expose request, tenant, export, state, and attempt identifiers without exposing the signed URL. A useful alert is a record stuck in a non-terminal state, because that catches a broken transition before a user reports a missing download.

The operational checklist belongs in the implementation record: authenticate, compare the export owner before rendering, resolve the approved region, write privately, verify readiness, sign briefly, and emit URL-free metrics. Keep cleanup tied to export retention. This makes the workflow reviewable when someone later adds bulk export, a second worker, or a new region.

## Can a Next.js API route keep a private PDF export tenant-safe?

An export record should own `tenantId`, `exportId`, its approved region, and its status. The request supplies an export identifier; the server resolves the rest. A client-provided tenant ID or bucket name is input, not authorization.

The flow is deliberately ordered: authenticate the caller, load the export record, compare ownership, resolve the server-side US or EU bucket, render tenant-scoped data, write a private object, confirm that the object is ready, and sign the read. If the record is missing or belongs to another tenant, return a normal not-found response before rendering. The object key helps inspection and retry behavior, but it is not the ownership check.

The main reliability invariant is idempotency. One `exportId` maps to one stable destination, so a request that times out after a successful write can converge on the same object. A signed URL is the last step, not the record of truth.

Here is a provider-neutral boundary. The storage adapter can sit behind any object store that supports private writes and signed reads, while the route keeps the authorization decision in application code.

```ts
import { NextRequest, NextResponse } from "next/server";

type Region = "us" | "eu";
type ExportRecord = {
  exportId: string;
  tenantId: string;
  region: Region;
  status: "queued" | "ready";
};

type ExportStore = {
  findExport(exportId: string): Promise<ExportRecord | null>;
  putPrivateObject(input: {
    bucket: string;
    key: string;
    body: Uint8Array;
    contentType: "application/pdf";
    idempotencyKey: string;
  }): Promise<void>;
  hasObject(bucket: string, key: string): Promise<boolean>;
  signRead(input: {
    bucket: string;
    key: string;
    expiresInSeconds: number;
  }): Promise<string>;
};

declare const exportsStore: ExportStore;
declare function authenticatedTenant(request: NextRequest): Promise<string | null>;
declare function renderTenantPdf(tenantId: string, exportId: string): Promise<Uint8Array>;

const buckets: Record<Region, string | undefined> = {
  us: process.env.EXPORT_BUCKET_US,
  eu: process.env.EXPORT_BUCKET_EU,
};

export async function POST(request: NextRequest): Promise<NextResponse> {
  const tenantId = await authenticatedTenant(request);
  if (!tenantId) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { exportId } = (await request.json()) as { exportId?: string };
  if (!exportId) return NextResponse.json({ error: "Invalid export" }, { status: 400 });

  const record = await exportsStore.findExport(exportId);
  if (!record || record.tenantId !== tenantId) {
    return NextResponse.json({ error: "Export not found" }, { status: 404 });
  }

  const bucket = buckets[record.region];
  if (!bucket) return NextResponse.json({ error: "Region is not configured" }, { status: 500 });

  const key = `tenants/${encodeURIComponent(tenantId)}/exports/${encodeURIComponent(exportId)}.pdf`;
  const body = await renderTenantPdf(tenantId, exportId);
  await exportsStore.putPrivateObject({
    bucket,
    key,
    body,
    contentType: "application/pdf",
    idempotencyKey: exportId,
  });

  if (!(await exportsStore.hasObject(bucket, key))) {
    return NextResponse.json({ error: "Export is still processing" }, { status: 202 });
  }

  const url = await exportsStore.signRead({ bucket, key, expiresInSeconds: 300 });
  return NextResponse.json(
    { url },
    { headers: { "Cache-Control": "private, no-store" } },
  );
}
```

The route has one intentionally visible configuration boundary: bucket names come from the environment, while the record chooses only an approved region. Production configuration should be validated before traffic arrives. For large PDFs, move rendering and writing to a worker; the same route can later sign an authorized record whose status is `ready`.

## Make recovery depend on evidence, not filenames

Retries are where a clean example meets an actual system. A queued record can be claimed by a worker, rendered, written privately, and marked ready only after the write is confirmed. If rendering succeeds but writing fails, the record needs a retryable failure state and an operator-visible reason. If writing succeeds but the database update fails, the stable key lets the next attempt inspect and reconcile the object instead of creating an untracked second file. A retry must not silently change the tenant, region, or export key. For example, if a worker dies after the object write but before the status update, the next attempt should compare the expected bucket, key, and export ID, then either adopt that exact object or report a deliberate reconciliation decision; it should never sign an object merely because a similarly named file exists. That extra check costs a database read, but it converts an ambiguous timeout into an inspectable state transition.

Keep the region in the record because it is policy data and part of the audit trail. Do not infer it from a browser preference or accept an arbitrary bucket from the request. If there is no region mapping, fail closed and repair the application record; silently choosing a default defeats the policy. US and EU placement also does not, by itself, define replication, retention, deletion, or disaster recovery. Those are separate controls.

The response that contains the URL should be `Cache-Control: private, no-store`. That protects the API response from being stored by shared caches; it does not revoke a URL that has already been copied. Do not log the signed URL, and be careful with browser history, analytics, and support tickets. A fresh request for a link should repeat authorization.

Keep it private.

Short windows help, but they are a policy choice. Five minutes is a concrete example in the route, not a universal setting. Pick a lifetime that covers normal user behavior without turning a download link into a durable credential.

I treat a `429` as a pacing signal. Retry only an operation whose idempotency behavior is understood, honor `Retry-After` when present, and cap attempts. I'm not sure one retry count can fit a queue, a render timeout, and an object write; your mileage may vary. Record the attempt count and export ID, never the capability URL.

## The cost and boundary of private export delivery

The catch is that this pattern is not suitable when the product needs permanent public URLs, public-read ACLs, static hosting, object versioning, WORM retention, or independently managed browser-upload CORS. Choose a boundary that explicitly provides the missing control, or add a separate service for that job. Do not loosen a private export policy to fit a convenient link.

There is also real application overhead: an export record, worker or route-timeout policy, lifecycle cleanup, and metrics. That is a poor fit for a throwaway non-sensitive image. It is a reasonable cost for tenant data when the team needs deletion and state inspection.

| Decision | Keep this flow when | Reconsider it when |
|---|---|---|
| Access | Each read must be authorized for one tenant | The asset is intentionally public |
| Placement | A tenant policy selects US or EU storage | One global location is acceptable |
| Delivery | A short-lived capability is enough | Users need stable, cacheable URLs |
| Recovery | The application defines retention and backups | Immutable history is a hard requirement |

The decision rule is simple: choose the private stateful flow when tenant isolation and controlled delivery matter more than a permanent link. Choose another boundary when the required storage guarantees are different. The important part is making that trade explicit before the first PDF reaches a browser.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://www.backblaze.com/cloud-storage/pricing
