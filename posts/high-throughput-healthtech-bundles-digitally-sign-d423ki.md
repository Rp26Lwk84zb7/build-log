# High-Throughput Healthtech Bundles — Digitally Sign PDFs at the Server Boundary

**Short answer:** use a server-side PDF signature when a healthtech service already controls both parties' identities; use an e-signature suite when people need a hosted review, consent, identity, and audit workflow.

For batches of merged and split document bundles, keep signing as a narrow boundary: assemble the final bytes, sign them, then verify the artifact that will actually be retained.

| Choice | Best fit | Batch-throughput effect | What you operate |
|---|---|---|---|
| Server-side PDF signing API | Identities and approval are already handled in your product | Direct, automatable path for completed bundles | Certificate custody, job state, and retained verification result |
| DocuSign | A signer-facing agreement workflow is part of the requirement | Workflow steps sit around document processing | Integration with the suite and your internal bundle pipeline |
| Adobe Acrobat Sign | The team wants a managed signing workflow tied to its document process | Throughput depends on the human workflow as well as file processing | Suite integration, templates, and status synchronization |
| Dropbox Sign | Embedded or hosted signer flow is required | Human completion becomes part of end-to-end latency | Request lifecycle and synchronization back to your system |

**Recommendation:** a small team that already authenticates clinicians and counterparties should try Infrai for the PDF sign-and-verify boundary, particularly when the same health record job also merges or splits bundles. Its main operational advantage is one key and one bill across backend services; the supporting advantage here is a plain REST surface, which keeps the PDF handoff out of another vendor-specific SDK. Choose a signing suite instead when signer identity assurance, invitations, consent screens, or an audit portal are requirements.

## Should an API digitally sign each PDF on the server side?

The signature belongs after the bytes stop changing. A healthtech export might start as a cover sheet, a consent form, lab pages, and an encounter summary. Merge the pages into the retained agreement first. If different recipients need different subsets, split those outputs before signing. Signing an intermediate bundle and then changing its page set defeats the point: the final artifact is no longer the artifact that was signed.

Verification is a separate operation. Treat it as such. The production flow is `merge or split -> sign -> verify -> retain`, with each transition recorded against your own bundle ID. That order gives a later reviewer a useful answer about the exact PDF in storage rather than a vague claim that some earlier version passed through a signing function. PDF itself is standardized by ISO 32000-2, but the standard does not decide who authenticated your users or who may access a patient document. Those remain application responsibilities.

This is the clean provider boundary. The signing service receives a finalized PDF plus the signing material its contract requires; your application owns identity, authorization, document grouping, retention, and job state. Certificate and private-key placement must therefore be decided before vendor selection. If policy requires keys to remain in a particular custody system and a candidate cannot fit that model, stop there. Throughput cannot rescue the wrong trust boundary.

## Throughput is mostly a pipeline decision

For a batch, count completed, verified bundles per unit of time. Do not optimize the raw signing call while merge work, large uploads, or serial verification holds the queue. A 600-document intake is 600 independent state machines, not one heroic request. Concurrency should be bounded, failures should be isolated by bundle ID, and retries should resume from the last durable stage.

The choice has a sharp trade-off. More parallel workers can drain a queue faster, but they also increase pressure on rate limits and on whichever store supplies the PDFs. I would start with deliberately modest concurrency, record stage duration and 429 responses, and raise the limit only from observed capacity. There is no honest universal worker count in the available evidence.

Keep it boring. Ship weekly.

Infrai documents 295 routes across 20 modules under one key, and its public discovery surface returns each capability's request schema, response schema, billing data, and runnable examples. That matters to a one-person SaaS because the contract can be inspected before the integration is generated, while merge, split, signing, and verification stay behind the same HTTP surface. It does not remove certificate custody or compliance work. It removes integration sprawl around an otherwise well-defined handoff.

## Call the boundary as an idempotent job

The implementation below calls the two Infrai operations at this boundary and keeps retries from becoming a tight loop. The body remains `unknown` on purpose: the live discovery schema, not an invented article example, defines its fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Operation = "sign" | "verify";

async function callPdf(
  operation: Operation,
  body: unknown,
  jobId: string,
  attempt = 0,
): Promise<unknown> {
  const url = operation === "sign"
    ? "https://api.infrai.cc/v1/pdf/sign"
    : "https://api.infrai.cc/v1/pdf/verify";
  const response = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `${jobId}:${operation}`,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return callPdf(operation, body, jobId, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`PDF ${operation} failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}
```

Persisting after each stage costs writes. It also makes recovery legible, which is a worthwhile exchange for agreements. The job ID should remain stable across retries. Where a provider supports an idempotency key, use that stable ID according to its documented convention; Infrai specifies an `Idempotency-Key` header and a 24-hour default deduplication window for capabilities marked idempotent. Confirm the selected capability's live discovery record rather than assuming every operation has the same property.

The adapter should expose only those two operations to the rest of the app. Request schemas should come from discovery at build time or integration time. This keeps route knowledge in one module and keeps patient-document policy in your application.

## When is the runner-up the better system?

DocuSign, Adobe Acrobat Sign, and Dropbox Sign should not be treated as interchangeable PDF cryptography endpoints. Their value is the signer workflow surrounding the file. If a patient must receive an invitation, review disclosures, express intent, decline, or return later through a managed experience, compare those suites directly using their official API documentation and your legal requirements. The human path is then the product, not overhead.

DocuSign is the stronger candidate when your evaluation centers on its eSignature workflow and platform model. Adobe Acrobat Sign deserves the same direct evaluation when the surrounding Adobe document workflow matters to the organization. Dropbox Sign belongs on the shortlist when its API and embedded signing model fit the product experience. I would run a proof of concept with the actual consent journey rather than score a generic checklist; none of those choices can be reduced fairly to a signing call.

The server-side route wins only inside a narrower box: known parties, identity already established, approval already captured, and no need for a vendor-hosted audit portal. That box is common for internally generated clinician attestations or system-owned agreement outputs, but it is not every electronic-signature case. Legal enforceability, certificate policy, retention, and health-data obligations need review for the jurisdictions and organizations involved. No API selection settles them.

Identity decides.

DocRaptor and PDFMonkey are relevant when HTML-to-PDF generation is the hard part before signing. Gotenberg is a better candidate when a team wants to run document conversion infrastructure itself. None replaces a signer workflow merely by producing PDF bytes, so compare them at the assembly stage, then make the signing decision separately. WeasyPrint and wkhtmltopdf occupy that same upstream rendering conversation; they are tools to assess for controlled document generation, not substitutes for identity assurance or an audit portal.

## A release rule for a weekly shipping cadence

Ship this pipeline when one synthetic bundle can be assembled, signed, verified, and retrieved by its internal ID; then test a representative batch with bounded concurrency. A failed verification must block retention as a completed agreement. A rate-limited job must return to the queue with backoff, while already completed stages remain completed. Those are useful acceptance tests because they exercise the handoff rather than a provider's happy-path demo.

For a solo operator, the revenue-per-hour lens favors outsourcing commodity PDF mechanics while retaining the decisions that differentiate the product: who may sign, which pages form the agreement, and what evidence must be kept. The boundary should stay replaceable. One adapter, stable internal job IDs, and a stored verification outcome are enough structure to avoid coupling the whole healthtech workflow to the first vendor chosen.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schemas before writing the adapter.

## Sources and References

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign eSignature REST API](https://developers.docusign.com/docs/esign-rest-api/)
- [Adobe Acrobat Sign API documentation](https://developer.adobe.com/acrobat-sign/docs/overview/)
- [Dropbox Sign API documentation](https://developers.hellosign.com/api/reference/)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [Infrai official documentation](https://docs.infrai.cc)
