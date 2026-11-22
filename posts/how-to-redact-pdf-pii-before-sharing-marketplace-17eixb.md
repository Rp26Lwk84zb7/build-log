# How to Redact PDF PII Before Sharing Marketplace Discovery Documents

| Template ownership | Working approach | Release condition |
| --- | --- | --- |
| Marketplace controls the source template | Omit unnecessary PII during generation, then inspect the resulting PDF | Approve the exact document that will leave the marketplace |
| Seller controls the uploaded document | Preserve the original privately and create a separately reviewed redacted derivative | Approve removal across the uploaded document, including scanned content |
| Ownership is mixed or unknown | Treat the document as an uncontrolled upload | Establish provenance before applying template-specific rules |

Template ownership determines how much sensitive content a marketplace can avoid putting into a discovery document in the first place. TL;DR: remove unnecessary PII at generation when you own the template; use a redaction workflow for seller uploads and any remaining sensitive content, then watermark only the approved derivative before external sharing. A watermark identifies the sharing context. It does not establish that PII has been removed.

For a solo SaaS founder, the useful unit of comparison is revenue per hour of engineering attention. Outsource the undifferentiated document processing if that fits the custody requirements, but keep the disclosure policy and template decisions inside the marketplace. Ship weekly. A weekly release should still have a way to stop one unsafe document from leaving.

## Who owns the template that introduced the PII?

Start with two document paths: a marketplace-generated transaction summary and a seller-uploaded supporting document. These are illustrative inputs, not a claim about an existing service. Both may enter a legal discovery workflow, and both may need a recipient watermark, but only the first gives the marketplace direct control over the fields that enter the initial PDF.

For the generated summary, define the external-sharing template separately from the operational template. Decide which fields belong in discovery with the person responsible for disclosure. Do not silently equate “personally identifying” with “must always remove”; legal relevance and disclosure scope require an explicit policy decision. If a seller contact field is unnecessary for the approved external version, omit it before rendering. That reduces the amount of downstream material that needs redaction review, although it does not prove the final document is safe.

Ownership changes the work.

For an uploaded document, the marketplace cannot assume its own template rules describe the file. A seller might supply a scanned page, a document with selectable text, or a mixture. Make the review cover every page and any attached material that the disclosure policy brings into scope. A coordinate rule for a known transaction summary should never silently become a rule for arbitrary seller evidence. The attractive shortcut is a shared “hide contact details” rectangle; the failure is applying it to a page whose layout has changed.

Keep the private original linked to the derivative through an internal record. Do not overwrite the original to make an external-sharing copy. This preserves the distinction between retained evidence and the artifact approved for release.

## How should an API redact PII from a PDF before sharing?

The second criterion is what the processing boundary promises. Ask for an applied redaction result with inspection evidence, rather than an instruction to draw a dark shape over the page. The PDF specification is the format reference; the acceptance contract must identify which document contents the selected processing method examines and removes. The standard's existence does not certify an implementation or make a particular output safe.

Define the check in terms of recoverability. For the approved scope, require reviewers to inspect both the visible pages and content obtainable through the agreed extraction and inspection tools. Include text associated with scanned pages, metadata, annotations, and attachments in that scope where applicable. These are inspection requirements, not a promise that a single text search proves complete removal. A visual check and a text extraction check answer different questions. Neither should substitute for reviewing what the document actually contains.

Watermark last.

For this marketplace scenario, the watermark can identify an approved recipient or disclosure batch without copying raw seller contact data into the overlay. Check the final watermarked bytes again: the externally shared artifact is the one that needs approval. A clean intermediate output is insufficient if the next step can add sensitive labels or change the artifact. Likewise, rendering quality matters because an unreadable disclosure document creates a different operational problem even when the intended fields have been removed.

## Implement the ownership decision before calling the processing API

The following TypeScript example runs locally and makes the routing decision explicit. It is a policy preflight, not a PDF redaction engine. Save it as `sharing-policy.ts` and run it with a TypeScript runner in your application environment. Its interfaces are local application types; they do not imply an existing vendor endpoint or supported API feature.

Step 1 is to record ownership and a template revision at intake. Step 2 is to derive the required processing and review stages. Step 3 is to refuse release until the final artifact has passed them. The example exercises all three steps using synthetic marketplace identifiers.

```ts
type Ownership = "marketplace" | "seller" | "unknown";
type Stage = "omit-fields" | "redact" | "inspect" | "watermark";

interface DocumentJob {
  documentId: string;
  ownership: Ownership;
  templateRevision?: string;
  disclosureBatch: string;
}

interface FinalReview {
  inspectedDigest: string;
  releaseDigest: string;
  scopeApproved: boolean;
  removalApproved: boolean;
  watermarkApproved: boolean;
}

function plan(job: DocumentJob): Stage[] {
  if (job.ownership === "marketplace") {
    if (!job.templateRevision) {
      throw new Error("Controlled templates require a revision");
    }
    return ["omit-fields", "redact", "inspect", "watermark", "inspect"];
  }
  return ["redact", "inspect", "watermark", "inspect"];
}

function requireReleaseApproval(review: FinalReview): void {
  if (
    !review.scopeApproved ||
    !review.removalApproved ||
    !review.watermarkApproved ||
    review.inspectedDigest.length === 0 ||
    review.inspectedDigest !== review.releaseDigest
  ) {
    throw new Error("Final artifact requires review before sharing");
  }
}

const job: DocumentJob = {
  documentId: "seller-evidence-042",
  ownership: "seller",
  disclosureBatch: "disclosure-017",
};

const stages = plan(job);
if (stages[0] !== "redact") {
  throw new Error("Seller uploads must use the uncontrolled path");
}
console.log({ documentId: job.documentId, stages });

let blocked = false;
try {
  requireReleaseApproval({
    inspectedDigest: "",
    releaseDigest: "",
    scopeApproved: false,
    removalApproved: false,
    watermarkApproved: false,
  });
} catch {
  blocked = true;
}
if (!blocked) throw new Error("Unreviewed output was allowed");
console.log("Unreviewed output blocked");
```

The judgment here is deliberately narrow: missing review evidence must stop release. The strings in the review record must come from trusted application state. In production, compute the artifact digest from the actual bytes using a cryptographic hash and associate approval with that digest. Matching caller-supplied strings is not proof of inspection. Keep reviewer authorization and the share operation behind the same application boundary, so an external caller cannot approve its own file.

Connect the plan to an internal processing adapter. That adapter should submit the private input, track completion, fetch the derivative, and return evidence for review under your chosen contract. Keep the job pending if processing fails, times out, or returns an incomplete result. Do not share the original as a fallback. Retries should reuse the logical job identity and avoid creating an accidental external delivery.

## Keep weekly releases small enough to review

Treat a template edit as a disclosure change when it can alter the external document. A new field, a moved label, or an extra page should trigger review of representative output. Use synthetic fixtures with known sensitive tokens and expected retained text. Include a controlled template revision, an uncontrolled upload, and a scanned input in the acceptance set. Those are proposed fixtures, not measured coverage guarantees.

One negative test earns its maintenance cost: approve a derivative, change its bytes, and verify that release is refused. Another should show that a failed processing job cannot reach the sharing operation. These checks protect the workflow boundary; PDF inspection still needs its own acceptance tests and human judgment where the disclosure policy requires it.

For observability, record the internal document identifier, template revision when present, processing state, and approval state. Keep document text and raw PII out of routine logs. Measure time spent reviewing and repairing derivatives alongside processing latency. A fast transformation that repeatedly demands manual cleanup can consume the founder's shipping week, while a slower batch process may fit the disclosure deadline perfectly well.

That is the revenue-per-hour trade: spend scarce attention on what the marketplace is allowed to disclose and on the evidence that the release gate works. Delegate repeatable processing under a testable contract.

## When is processing every document the better choice?

The runner-up is a uniform derivative workflow that treats even marketplace-generated PDFs as uncontrolled inputs. It is a reasonable choice when templates change frequently, historical documents lack reliable revision records, or the incoming discovery batch mostly consists of seller uploads. It gives up some opportunities to prevent PII from entering the document, in exchange for fewer routing assumptions to maintain. The template-first approach has a limitation: it requires reliable ownership and revision records. It is not suitable for an archive whose provenance cannot support those decisions. The trade-off is extra processing and review for a simpler routing policy. For example, consider an illustrative batch of 12 documents with only 2 generated from a controlled template. Maintaining a special path for those 2 documents may cost more engineering attention than it saves in review. These numbers describe a planning example, not a benchmark; use the actual document mix to decide whether the extra branch earns its maintenance cost.

Don't guess ownership.

Choose it when that simpler operating model fits the actual document mix. Either path still needs approved disclosure scope, applied removal, inspection of the final watermarked artifact, and a release decision tied to its bytes. Template ownership determines the first step; it never replaces the last check.

## References

The format reference below establishes the PDF specification. The ownership matrix, example identifiers, processing plan, and release conditions above are proposed application design choices, not capabilities asserted for a particular implementation. Validate the processing contract against representative files before external sharing.

## Sources

- ISO 32000-2 — Portable Document Format: https://www.iso.org/standard/75839.html
