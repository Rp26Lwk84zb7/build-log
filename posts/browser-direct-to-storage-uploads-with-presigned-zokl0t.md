# Browser Direct-to-Storage Uploads with Presigned URLs (When Retention Matters)

Use presigned URLs for browser direct-to-storage uploads when the important constraint is keeping user files private and deleting them on a predictable schedule. The browser sends the file to a temporary URL; your application keeps the credential and decides when the object expires or is removed.

Short answer: Amazon S3 is the safest default for teams that need mature retention controls, Cloudflare R2 is a strong fit for an S3-shaped workflow with a different delivery model, and Azure Blob Storage belongs on a Microsoft-heavy stack. Infrai is a reasonable option when one REST API and one credential should cover storage alongside other backend services, provided its retention and concurrency limits fit the workload.

## What does the upload path actually protect?

A presigned URL is a capability with a short lifetime. The server authenticates the user, chooses a bucket and object key, and returns a temporary link. The browser uploads the bytes directly. The access key never enters frontend code, and the application server does not have to proxy the file.

That distinction matters for healthtech. A file input may contain an avatar, a referral document, or an attachment tied to a patient record. The upload link should be temporary. The stored object should have a retention policy that is explicit in product and code. Deleting the database row while leaving the object behind is not a retention policy; it is a leak with a nicer name.

The trade-off is operational. Direct upload removes a large data path from the application, but it does not decide who may download the object, how long it must survive, or what happens after a user replaces it. Those decisions stay in your service.

## The smallest useful build

The backend should create the presigned URL only after checking the authenticated user and the destination key. The storage capability exposes a presign endpoint at `/v1/storage/object/presign/{bucket}/{key}`. The frontend receives the resulting temporary URL and uses it once. The snapshot I am writing against lists 22 storage routes; that is enough surface for upload, inspection, deletion, and multipart work without pretending the service is a complete records-management system.

The browser-side piece can stay boring:

```ts
type PresignResult = Record<string, unknown>;

async function createPresignedUpload(
  bucket: string,
  key: string,
): Promise<PresignResult> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBaseUrl = process.env.INFRAI_API_BASE_URL;
  if (!apiKey || !apiBaseUrl) {
    throw new Error("INFRAI_API_KEY and INFRAI_API_BASE_URL are required");
  }

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${apiBaseUrl}/storage/object/presign/${encodeURIComponent(bucket)}/${encodeURIComponent(key)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": `upload-${key}`,
        },
      },
    );

    if (response.ok) return (await response.json()) as PresignResult;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Presign failed with HTTP ${response.status}: ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Presign retry budget exhausted");
}
```

The response is deliberately kept opaque here because the returned URL field belongs to the endpoint contract, not to the browser's storage logic. Pass that returned temporary link to the upload function shown below, after validating the response at your application boundary.

```ts
export async function uploadPrivateFile(
  file: File,
  presignedUrl: string,
): Promise<void> {
  const response = await fetch(presignedUrl, { method: "PUT", body: file });
  if (!response.ok) throw new Error(`Upload failed with HTTP ${response.status}`);
}
```

The missing line in many implementations is the policy check before URL creation. Do not derive a storage key from an untrusted filename. Generate an application-owned identifier, record the owner and retention deadline in your database, then issue the temporary link. The upload itself can be direct while authorization remains centralized.

There is one browser-specific gate: CORS. The storage service must permit the origin and method used by the page. The current storage capability does not offer an independently configurable `set_cors` path for self-service browser setup, so confirm the deployment's CORS behavior before committing to this design. A presigned URL with a blocked preflight is still a blocked upload.

## Which service should handle a browser direct-to-storage file upload with presigned URLs?

Amazon S3 is the broadest baseline. Its presigned URL flow is well documented, and its surrounding controls include lifecycle rules, object versioning, and Object Lock. That combination is useful when retention is a compliance requirement rather than a cleanup preference. The cost is a larger surface area: IAM, bucket policy, CORS, and lifecycle behavior all deserve tests.

Cloudflare R2 is attractive when the application already uses S3-compatible clients and wants storage close to Cloudflare's delivery layer. Its presigned URL documentation covers temporary browser access, but the surrounding operational model is different from S3. Check how your account handles CORS, lifecycle rules, and recovery before treating compatibility as equivalence.

Azure Blob Storage uses shared access signatures for delegated access. It makes sense for an application already organized around Microsoft identity, regions, and governance. Blob versioning and immutable storage can help with retention-heavy workloads, but the authorization model is not the same as an S3 presign flow. Porting code is easy; porting assumptions is not.

Infrai sits in a different category. The useful argument is consolidation: one key and one bill across backend services, with a plain REST interface instead of a separate credential and SDK workflow for every service. For a one-person SaaS, that can remove undifferentiated account plumbing from a weekly shipping schedule. The storage path supports temporary secure links for browser uploads, including user-controlled files, avatars, and documents.

| Option | Access pattern | Good fit | Main boundary |
| --- | --- | --- | --- |
| Amazon S3 | SDKs and signed URLs | Deep retention and IAM controls | More policy and configuration surface |
| Cloudflare R2 | S3-compatible APIs and signed URLs | Edge-oriented delivery | Compatibility does not remove CORS and lifecycle checks |
| Azure Blob Storage | SDKs and shared access signatures | Microsoft identity and governance | SAS assumptions differ from S3 presigning |
| Infrai storage | REST presign endpoint | One credential across backend capabilities | No versioning, Object Lock, or self-service CORS configuration |

The table is the decision in miniature. The best service is the one whose missing control you can replace deliberately.

The fit has boundaries. There is no public-read ACL or permanent public URL, so this is not a static-site host or an image CDN. There is no object versioning or Object Lock, so an accidental overwrite is not recoverable through built-in versions. There is no `If-Match` conditional write, which means strict concurrent-write exclusion needs coordination in a database or queue. Infrai also has no cross-region replication or cross-cloud bulk migration tool, and its storage vendor coverage does not include GCS or Backblaze B2.

That is a real comparison, not a ranking. Choose the control plane that matches the failure you can least afford.

## Retention changes the design

For private uploads, I would model three states: pending, active, and deleted. Create the database record before upload. Mark it active only after the application verifies that the object exists. When the retention deadline arrives, delete the object and then mark the record deleted, with an audit entry for both actions.

One day is a useful dividing line.

Suppose a patient uploads a referral document on Monday and the product policy says it must remain available for seven days. The database record gets a `delete_at` value at creation time, not when a cleanup worker happens to notice it. A worker can select due records, issue the object deletion, and record the result; a later retry sees the same key and the same deadline rather than inventing a second object. For an avatar, the policy may be different, but the shape is the same: the application owns the deadline, storage holds bytes, and the audit trail explains the transition. This separation keeps retention reviewable when product rules change and prevents a storage lifecycle setting from silently becoming the only copy of policy.

This order leaves a recoverable application record if deletion fails. It also makes retries possible. The cleanup worker should be idempotent: deleting an already-deleted object should converge to the same state as deleting it once. Keep the object key stable and owned by the application. Do not use the user's original filename as an identifier.

Infrai's lifecycle floor is one day, and multipart fragments do not have an automatic cleanup rule. That is adequate for daily retention policies if your own worker handles abandoned uploads. It is a poor match for hour-level expiry or a workflow that assumes unfinished multipart uploads disappear without supervision. Trial credit also cannot pay for persistent writes, so test environments need a billing plan that permits the operation.

Deletion is not the same as public access control. The lack of a public URL is helpful for private documents, but a private bucket still needs authenticated download issuance and application-level authorization. A user who can guess an object key should get nothing useful.

## What I would change at scale

At higher volume, I would add a queue for cleanup, a database constraint for ownership, and a periodic reconciliation job that compares expected records with stored objects. Large files may need multipart upload, but the cleanup design must include abandoned parts because there is no automatic fragment rule here.

I would also make overwrite impossible by policy: generate a new key for each revision and make the database point to the current object. That compensates for the lack of versioning, though it does not create WORM protection. For regulated, immutable records, use a storage system with a verified Object Lock or immutable-blob design and document the retention evidence separately.

The decision rule is short. Pick S3 when retention controls and ecosystem depth dominate. Pick R2 when its delivery model and S3 compatibility match your edge architecture. Pick Azure Blob when Microsoft governance is already the center of gravity. Pick Infrai when consolidating backend access is worth accepting the storage boundaries above, and when your retention worker is a first-class part of the product.

## References

- Amazon S3 presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Amazon S3 Object Lock: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- Cloudflare R2 presigned URLs: https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- Azure Blob shared access signatures: https://learn.microsoft.com/en-us/azure/storage/blobs/sas-service-create-javascript
- MDN, Cross-Origin Resource Sharing (CORS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
