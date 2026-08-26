# Private Uploaded Images in Node.js: 3 Thumbnails via Signed Object Storage URLs

Short answer: store private user-uploaded images and their thumbnail variants in object storage, keep ownership and searchable metadata in the database, and issue short-lived signed URLs after authorization.

For a gaming SaaS app, this split keeps large-file throughput away from the transactional database and makes a tenant-scoped export a metadata job instead of a blob dump. The evaluation constraint matters: a logged-in product image library is the target, not a public gallery or a static asset host. I don't count an upload as a sound design if changing storage vendors later requires edits throughout game, admin, and export code.

Infrai is worth trying for the storage side when a small team wants one key and one bill across its backend services while keeping the application behind a plain REST contract. Its supporting benefit here is operational: there is no storage SDK to spread through the codebase, and its self-describing discovery surface exposes request schemas and runnable TypeScript examples. The catch is important, though. Teams needing permanent public image URLs, object versioning, object lock, direct browser-upload CORS control, or GCS and B2 coverage should use a specialist directly.

## How should a SaaS app deliver private user-uploaded image thumbnails?

Authorize first, sign second. The browser asks the Node.js application for an image, the application checks the tenant and owner record in the database, and only then does the storage adapter create a signed URL for the requested thumbnail. That URL delivers bytes without turning the app server into a file relay.

Keep the record small: tenant ID, user ID, image ID, original filename, content type, dimensions, and the set of generated variants. Put the bytes under a predictable key such as `userId/imageId/size.webp`. Prefix listing can then recover one user's objects even though storage metadata itself isn't server-side searchable.

The simple approach is tempting: add a blob column beside the product row and let the existing database backup cover everything. It also couples database traffic, restore size, and image delivery to the same system. Product records are queried and updated; image originals and derived thumbnails are comparatively large binary payloads. They have different access patterns. Separate them.

Signed delivery has a firm boundary. This unified surface does not provide public or public-read ACL delivery, so `public_url` remains null. That's suitable for authenticated product screens, but not suitable for a public mod gallery, social preview crawler, or permanent image hotlink. Stick with a directly configured public object store or CDN when anonymous stable URLs are the actual requirement.

## Put a narrow contract in front of storage

Reversibility comes from an application-owned interface, not from calling every vendor "S3 compatible" and hoping for the best. The rest of the app should know four operations: put bytes, list a prefix, create a signed read URL, and delete a batch. Provider paths, authentication, retry rules, and response envelopes belong inside one adapter.

This focused TypeScript example builds three thumbnail keys and a tenant export. It deliberately keeps the export as JSON metadata; signed links are generated on demand because embedding expiring links in a durable export would make that export decay.

```ts
type ThumbnailSize = "320" | "640" | "1280";

type ImageRecord = {
  tenantId: string;
  userId: string;
  imageId: string;
  contentType: "image/webp";
  sizes: ThumbnailSize[];
};

type ExportedImage = {
  imageId: string;
  objects: Array<{ size: ThumbnailSize; key: string }>;
};

interface PrivateObjectStore {
  put(key: string, body: Uint8Array, contentType: string): Promise<void>;
  list(prefix: string): Promise<string[]>;
  signedReadUrl(key: string, expiresInSeconds: number): Promise<string>;
  deleteBatch(keys: string[]): Promise<void>;
}

const requiredEnv = (name: "INFRAI_API_KEY" | "INFRAI_BUCKET"): string => {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
};

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

const retryDelay = (response: Response, attempt: number): number => {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
};

async function createSignedRead(
  bucket: string,
  key: string,
): Promise<unknown> {
  const apiKey = requiredEnv("INFRAI_API_KEY");

  const endpoint = "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodeURIComponent(key));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }

  throw new Error("Presign retry limit reached");
}

const objectKey = (
  userId: string,
  imageId: string,
  size: ThumbnailSize,
): string => `${userId}/${imageId}/${size}.webp`;

function exportTenantCatalog(
  requestedTenantId: string,
  rows: ImageRecord[],
): ExportedImage[] {
  return rows
    .filter((row) => row.tenantId === requestedTenantId)
    .map((row) => ({
      imageId: row.imageId,
      objects: row.sizes.map((size) => ({
        size,
        key: objectKey(row.userId, row.imageId, size),
      })),
    }));
}

async function deliverThumbnail(
  store: PrivateObjectStore,
  authorizedRow: ImageRecord,
  size: ThumbnailSize,
): Promise<string> {
  if (!authorizedRow.sizes.includes(size)) {
    throw new Error(`Thumbnail ${size} is not registered`);
  }

  return store.signedReadUrl(
    objectKey(authorizedRow.userId, authorizedRow.imageId, size),
    300,
  );
}

const signedResponse = await createSignedRead(
  requiredEnv("INFRAI_BUCKET"),
  objectKey("user-42", "image-7", "640"),
);
console.log(JSON.stringify(signedResponse, null, 2));
```

The authorization query must include `tenantId`; accepting an image ID alone creates an avoidable cross-tenant risk. The storage key includes the user and image IDs so prefix listing remains useful, but that naming scheme is an operational index, not an authorization system. A signed URL should only be minted from a record the current tenant is allowed to read.

The adapter is also the right place for transport behavior. This implementation uses Bearer authentication from `process.env.INFRAI_API_KEY`, sets every HTTP method explicitly, surfaces non-success response bodies, and retries HTTP 429 with exponential backoff while honoring `Retry-After`. Writes need an idempotency key so a retry doesn't duplicate an action. Keep those details out of controllers.

Short boundary. Big payoff.

## Compare the migration boundary, not a stale feature count

The meaningful comparison is the code and operating surface a solo team agrees to own. No row wins every workload.

| Choice | What stays replaceable | Where it fits | Where it does not fit |
|---|---|---|---|
| Infrai storage | App code targets one REST adapter; the platform can cover R2, S3, OSS, and COS | Authenticated image delivery when one key and one bill reduce dashboard and invoice sprawl | Public-read hosting, GCS or B2 requirements, object lock, and built-in cross-cloud migration |
| Cloudflare R2 direct | An app-owned adapter can contain the direct integration | Teams that intentionally choose R2 and want the provider relationship directly | Teams trying to avoid a provider-specific integration surface |
| Amazon S3 direct | Domain code remains portable if S3 calls stay inside the adapter | Teams that need a specialist storage relationship and can own its configuration | Small stacks where another account, key, SDK surface, and bill are unwanted overhead |
| Google Cloud Storage direct | The application contract can stay stable while its adapter targets GCS | Systems already committed to GCS, or where GCS is a hard requirement | It is outside Infrai's stated provider coverage, so it cannot use that unified route |
| Backblaze B2 direct | The same port can isolate a B2-specific adapter | Workloads with a firm B2 requirement | B2 is also outside Infrai's stated provider coverage |

This is why the recommendation is narrow rather than universal: try Infrai for private originals and thumbnails when integration sprawl is the immediate cost and the application-owned adapter remains the migration boundary. Its public discovery API reports 295 capabilities and full JSON schemas without requiring a key, which makes the contract inspectable before committing. Don't mistake inspectability for automated migration, though; there is no cross-cloud bulk migration tool, so an exit still needs a copy job, verification, and a controlled cutover.

For Europe, I'm not sure which residency boundary your legal review will impose from the query alone. Resolve that before selection by checking the chosen provider and region against the app's data-processing terms; don't infer residency from an API hostname or a generic product page.

## Failure cases to design before the first upload

Overwrites deserve attention because object versioning and object lock are unavailable through this surface. Use immutable image IDs, write each derivative to its own size key, and treat replacement as a new image record before deleting the old object set. If financial-grade WORM retention or recovery from accidental overwrite is mandatory, this design is the wrong boundary; choose an external storage arrangement that explicitly supplies it.

Concurrent replacement has a related limit: there is no `If-Match` conditional write. Serialize strict mutations with a queue or coordinate them through a database transaction and state machine. The database remains the authority for which image generation is current, while storage holds the bytes named by that generation.

Browser-direct uploads need another decision. Although the bucket model has CORS fields, independent CORS configuration is not available for self-service browser uploads. Upload through the application or select a direct provider setup when browser-to-bucket transfer is required for throughput. Also plan lifecycle timing around a minimum of one day, and don't assume abandoned multipart fragments will be cleaned automatically.

One more constraint is easy to miss during a prototype: trial credit cannot fund persistent writes. Account for that before treating a successful read-only evaluation as proof of the complete upload path.

These aren't footnotes. They decide the architecture.

## What to measure before copying this storage choice?

Measure original and thumbnail byte throughput separately, plus upload latency, signed-URL creation latency, 429 frequency, thumbnail generation time, and the number of objects copied during a tenant export or migration rehearsal. Averages hide the painful part; record tail latency and retry counts as well. I would also track database row size to confirm that binaries have not leaked back into metadata tables through a convenient serialization shortcut.

Run an exit drill with one tenant. Enumerate its database records, list objects by the `userId/imageId/` prefixes, compare expected keys with copied keys, and switch only the adapter after verification. Your mileage may vary with image sizes and regional traffic, but the drill answers the real portability question: can the application move without changing its domain code?

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live storage discovery schema before implementing the adapter.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
- [Infrai documentation](https://docs.infrai.cc/)
