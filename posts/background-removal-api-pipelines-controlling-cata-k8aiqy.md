# Background Removal API Pipelines: Controlling Catalogue Storage and Cache Churn

For an ecommerce catalogue, the simplest workable background-removal pipeline is an asynchronous job that keeps one immutable source photo and one approved transparent master, then derives delivery sizes from that master. Do not cache every intermediate response. Cache by content hash plus transformation policy, and replace a cutout only after validation passes.

**Short answer:** choose the API behind a small adapter, but make the storage contract the real decision. A clean cutout is useful once. A reproducible cutout that does not multiply objects on every retry is useful for years.

| Choice | Stored objects per accepted photo | Cache behavior | Best fit |
|---|---:|---|---|
| Synchronous request in the upload path | Source plus result | Easy to duplicate on retries | Tiny catalogues with manual review |
| Queued job with a content-addressed master | Source plus approved master | Stable key across retries | Most growing catalogues |
| Reprocess every delivery request | Source only at rest | Repeated compute and cold misses | Short-lived or rarely viewed inventory |

My default is the queued job. It keeps the upload path short, makes retries idempotent, and separates vendor choice from catalogue data. For a solo SaaS, that boundary matters: I want to ship weekly, and background segmentation is undifferentiated infrastructure.

## How should an ecommerce product image background removal API handle storage?

Background removal changes the asset contract. The input is commonly an opaque photograph; the useful output needs an alpha channel so the catalogue can place the object on different backgrounds. The browser format guide from MDN documents which image formats support transparency. That makes format selection an architectural decision, not a cosmetic export setting.

Store the original bytes unchanged. They are the only input that lets you run a better policy later without compounding compression or edge errors. Next, store one approved cutout at the highest delivery resolution you need. Generate thumbnails from that approved master rather than asking the removal service to process the same photograph once per card size.

This is the first criterion: **one logical transformation should create one durable master**. If an API exposes many output sizes, that convenience can quietly turn into a larger object inventory. A 12-image responsive set across 40,000 SKUs is 480,000 derived objects before revisions. Add one revised photograph for a quarter of those SKUs and the asset graph grows again unless superseded derivatives have a deletion policy. The arithmetic is illustrative, not a benchmark, but it exposes the operational shape: output options that look harmless in an API response become persistent storage classes, cache entries, invalidation work, and backup inventory. Count those consequences before treating another size as free.

The second criterion is deterministic replay. A request should be identified by the source content, the removal policy, and the adapter version. Filename alone is weak because merchants replace images without renaming them. A timestamp is worse; every retry becomes new work.

## Use two keys, not one

I use separate identities for source bytes and transformation intent. The source hash answers, "Have I seen these exact bytes?" The policy key answers, "Have I already produced the output this catalogue expects?" Keeping them separate lets a policy change create a new master while identical retries converge on the existing one.

The key should include only inputs that can change pixels. Do not include a queue attempt number, request time, or worker hostname. Those values describe execution, not the image.

Keep it boring.

```ts
import { createHash } from "node:crypto";

type RemovalPolicy = {
  outputFormat: "png" | "webp";
  edgeMode: "standard" | "fine";
  policyVersion: number;
};

type RemovalClient = {
  removeBackground(input: Uint8Array, policy: RemovalPolicy): Promise<Uint8Array>;
};

type ObjectStore = {
  exists(key: string): Promise<boolean>;
  put(key: string, body: Uint8Array, contentType: string): Promise<void>;
};

const sha256 = (value: Uint8Array | string): string =>
  createHash("sha256").update(value).digest("hex");

async function buildCutout(
  input: Uint8Array,
  policy: RemovalPolicy,
  client: RemovalClient,
  store: ObjectStore,
): Promise<string> {
  const sourceHash = sha256(input);
  const policyHash = sha256(JSON.stringify(policy));
  const key = `cutouts/${sourceHash}/${policyHash}.${policy.outputFormat}`;

  if (await store.exists(key)) return key;

  const output = await client.removeBackground(input, policy);
  const contentType = policy.outputFormat === "png" ? "image/png" : "image/webp";
  await store.put(key, output, contentType);
  return key;
}
```

This adapter deliberately says nothing about an HTTP route, authentication scheme, or provider response. Those belong in the `RemovalClient` implementation. The catalogue worker only knows that bytes and a policy produce bytes. Swapping the implementation does not require renaming every stored asset or changing product records.

There is one catch: `exists` followed by `put` is not a distributed lock. Two workers can perform the same transformation concurrently. The final key still converges if object replacement is atomic, but compute may be duplicated. If duplicate work is material, claim jobs using the queue or metadata store before calling the API. Do not encode lock state into the asset key.

## Validate before the cache can spread a mistake

A successful API response is not the same as a usable catalogue image. Validate the decoded output before publishing its URL. Check that its format matches the declared content type, its dimensions meet the catalogue requirement, and it contains transparency when transparency is required. Keep the previous approved master live until the candidate passes. A structurally valid file can still be a bad product asset, so the promotion boundary should include both machine checks and the catalogue's review rule. Put the candidate under a temporary key, decode it with the same image library used by downstream workers, and promote only the key that passed. If validation fails, retain the source and the previous approved master, record the category, and let the queue decide whether another attempt makes sense. This avoids the nastiest cache failure in the design: distributing a valid URL whose pixels should never have become canonical.

Visual quality needs a small review set too. Include pale products on pale backgrounds, reflective packaging, holes between handles, fine hair-like edges, and products with intentional shadows. The exact set should resemble the inventory, not a generic demo gallery. Automated checks catch corrupt or structurally wrong files; representative review catches plausible but commercially bad cutouts. Run that set before an adapter or policy version becomes eligible for new jobs, then keep the old policy available long enough to compare candidates without overwriting approved assets.

Bad edges sell nothing.

Retries are different.

Retry transport failures through the queue, with a bounded attempt policy, but do not publish partial bytes. Record source hash, policy hash, output key, status, and failure category. Avoid logging image bytes or signed asset URLs. The useful operational question is which stage failed, not what the customer uploaded.

Only promote the candidate after validation. Then invalidate catalogue pages or image manifests that point to the old approved key. Immutable asset URLs can have long cache lifetimes because a new result gets a new key; the small mutable manifest needs shorter caching. This split prevents one correction from purging every thumbnail at the edge.

## When is the runner-up better?

The synchronous upload-path design is reasonable when the catalogue is small, an operator already waits for every upload, and immediate visual review is part of the workflow. It has fewer moving parts. The trade-off is that API latency and retries now delay the merchant action, so set a hard timeout and preserve the original even when removal fails.

Reprocessing at delivery time can fit temporary inventory where most products are never viewed. It minimizes durable derivatives, but it moves transformation latency and failure into the customer request path. Use it only when the cache can absorb repeat views and a missing cutout has a defined fallback.

There is also a case for storing more than one master: print and web workflows may have genuinely different resolution or color requirements. Make those explicit policies. Do not let arbitrary UI sizes become masters by accident.

The revenue-per-hour test is plain.

If maintaining custom segmentation does not improve the product's differentiator, outsource it behind the adapter. Spend engineering time on acceptance tests, idempotency, and review flow instead. Those pieces protect the catalogue regardless of which API processes the pixels.

## A compact decision rule

Choose the queued, content-addressed path when images live longer than a release cycle or appear at several delivery sizes. Choose synchronous processing when human review is already blocking the upload. Choose request-time processing only for short-lived, cold inventory with a safe fallback.

Measure object count, stored bytes by asset class, cache hit rate, duplicate job claims, validation failures, and time from upload to approved master. Those signals reveal whether the system is paying for useful catalogue assets or for retries and forgotten derivatives.

The simplest API is the one you can replace without rewriting the catalogue. Keep the source, approve one transparent master per policy, and derive the rest. That is a small system a one-person team can operate while still shipping features every week.

## Further reading

- MDN, “Image file type and format guide”: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
