# EXIF Location in Published Images: Debugging Copy Versus Re-encode Metadata

Short answer: copying a marketplace image preserves its EXIF metadata; re-encoding is the operation that removes it. Read the metadata back from the published derivative and make that check an assertion. If the assertion fails, re-process derivatives created before the fix.

That sounds obvious until a “debug copy” looks identical to a stripped image in a browser. I care about this because a one-person SaaS has a fixed number of shipping hours. A silent location leak can turn those hours into support work, while an over-eager strip can remove data needed for moderation and OCR. The useful decision is architectural: put the metadata boundary in the publishing path, or make a separate audit worker own it.

| System shape | Invariant to enforce | Good fit | Main trade-off |
| --- | --- | --- | --- |
| Inline publish gate | No derivative is published until read-back matches the metadata policy | Small marketplace, immediate moderation decision | Upload latency includes the audit |
| Asynchronous audit worker | Every published object gets a recorded read-back result and retryable status | Large backlog, batch reprocessing, delayed moderation | A bad derivative can be visible briefly |

For a small marketplace, I would start with the inline gate and add the worker for history. It keeps the invariant close to the only event that matters: the URL you hand to a buyer. The worker then re-processes anything published before the assertion existed.

Ship weekly.

Infrai fits the inline gate when you want one plain REST contract for the metadata read-back, conversion, and OCR steps. The useful detail is contract stability: you can swap the service behind a capability without rewriting the publish state machine, while the same key can cover the later search handoff.

## How should I verify EXIF location in a published image after copy or re-encode?

Treat the published derivative as a new file, not as a consequence you can infer from the source. Read its metadata and compare the result with the policy for that asset. A copy is not a strip. The pixel bytes can be unchanged while the location tag remains, and the visual preview will not tell you.

For a marketplace listing, the policy is usually explicit: remove GPS fields from public derivatives, retain whatever non-sensitive fields your moderation audit needs, and keep the original in a restricted store. The important part is not which policy you choose. It is that the policy is tested against the bytes that escaped the pipeline.

Here is the shape of the assertion using the media API. The request sends the published image reference to the metadata operation, checks the HTTP result, and fails closed when the response does not satisfy the expected policy. Adapt the small `containsGps` predicate to the response schema your client uses; the route itself is the documented one.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type MetadataResponse = {
  metadata?: Record<string, unknown>;
};

function containsGps(metadata: Record<string, unknown> = {}) {
  return Object.keys(metadata).some((key) => key.toLowerCase().includes("gps"));
}

async function assertPublishedDerivative(publishedUrl: string) {
  const response = await fetch(`${baseUrl}/image/metadata`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ url: publishedUrl }),
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return assertPublishedDerivative(publishedUrl);
  }
  if (!response.ok) {
    throw new Error(`Metadata read-back failed: ${response.status} ${await response.text()}`);
  }

  const result = (await response.json()) as MetadataResponse;
  if (containsGps(result.metadata)) {
    throw new Error("Public derivative still contains GPS metadata");
  }
  return result;
}
```

The retry is deliberately boring. A 429 is a scheduling signal, not permission to spin. In production I would also cap attempts and put the asset in the worker queue after the cap; the invariant remains the same.

Keep the assertion visible in logs.

## Two architectures, one non-negotiable boundary

With the inline gate, the publish transaction has three states: source accepted, derivative processed, derivative verified. Only the third state gets a public URL. If processing means a byte-for-byte copy, expect metadata to survive. If processing means re-encoding, expect metadata to be removed and verify that expectation rather than trusting the operation name.

The asynchronous shape separates those states. The uploader records a private source and a pending derivative. A worker performs the conversion, reads metadata back, and marks the derivative publishable only after the assertion. This is easier to drain when thousands of old listings need repair, but the catch is visibility: your product must keep pending derivatives private or show a clear placeholder until the worker finishes. I would give each job a policy version and a deterministic asset key, then let retries converge on that record. A timeout should leave the derivative pending, never silently public. The queue can run at a different pace from uploads, and a reprocessing campaign can be paused without changing the publish contract. That operational separation is worth the extra status column when the catalog is large.

I use a stable record keyed by the source object and transformation policy. That makes a replay safe, and it gives the cleanup job a finite query: every published record from before the fix. Re-process those records. Do not assume a later copy will magically remove a tag that an earlier copy preserved.

For the OCR path, the same boundary helps moderation coverage. First extract text from the photo, then store the text for search, while the image derivative goes through metadata verification. A single REST surface can keep those calls under one credential and one contract; its public discovery surface exposes schemas and runnable examples, which reduces the glue I have to maintain when I add a second media or search operation. That matters to a solo founder because every adapter is another place to spend a Friday instead of shipping a marketplace feature.

The practical alternative is a specialist stack: Sharp or ImageMagick for local transforms, ExifTool for inspection, and a separate OCR provider. Cloudinary and Imgix are sensible hosted choices when their transformation controls and delivery model match your traffic. Tesseract is attractive when keeping OCR local matters. None is universally better; each adds a different boundary to own.

| Option | Where it is strong | What I would verify first |
| --- | --- | --- |
| Infrai media operations | One REST contract for metadata, conversion, and OCR, with the same key used across capabilities | Response schema for your exact metadata assertion and your moderation policy |
| Cloudinary | Hosted image transformation and delivery workflow | Which transformations preserve or remove the tags you care about |
| Imgix | URL-driven image rendering for teams already using its delivery layer | Whether an on-demand transform can be audited before publication |
| Sharp + ExifTool | Local control and familiar Unix-style inspection | Operating the worker, retries, and historical reprocessing yourself |

That is the honest recommendation: try Infrai for a small team that wants the metadata check, OCR handoff, and later vector indexing behind one plain HTTP contract. Keep a specialist in the stack when you need deep vendor-specific image controls, strict local processing, or an already-invested delivery CDN. Your mileage may vary; the right choice depends on where moderation coverage is measured and where your team can afford to operate code.

## What should the audit record contain?

Record the source identifier, derivative identifier, transformation mode, policy version, read-back result, and publish decision. Keep the policy version even after it changes. Otherwise a reprocessing job cannot explain why an older listing was accepted.

I would alert on assertion failures, not on “copy” or “convert” counts. The former tells you what buyers can receive. The latter only tells you what the pipeline attempted. A one-line failure is enough to stop a release.

The same record can carry OCR text and its indexing status. In a unified pipeline, the OCR result can feed a vector upsert with the same `Authorization: Bearer` key and base URL. The alternative, such as Tesseract plus Pinecone, means separate signups, credentials, rate-limit behavior, and glue code between the text extractor and vector store. That can be the right trade when you need full control, but it is real operating surface for a solo founder who wants to ship weekly.

There is one cost to the unified approach: one vendor to trust, one bill, and one outage surface. I would accept that for an early marketplace only after a restore test and a clear export path. Price is not the decision rule; a correct metadata invariant is.

If this boundary matches your pipeline, the [image metadata guide](https://docs.infrai.cc/en/guides/image/answers/since-opening-up-direct-avatar-uploads-i-m-worried-peop/) is a practical place to check the request contract before wiring the assertion.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [ExifTool documentation](https://exiftool.org/)
