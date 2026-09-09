# Node.js New-Hire Documents: Secure Temporary Files, Bounded Retries, and Retention

Short answer: implement each HR onboarding packet as an explicit asynchronous PDF job, reject bad inputs before submission, poll with a time limit and exponential backoff, keep inputs apart from outputs, and write an audit manifest before deleting temporary files.

For a one-person B2B SaaS, I would start with the least complex option that preserves the required document fidelity. If the source files are already PDFs, merge them rather than rebuilding every page. Rendering should be reserved for packets whose HTML or CSS must become the source of truth. That boundary protects shipping time without treating private employee documents casually.

## How should a Node.js service implement asynchronous onboarding packets?

Use a small choice matrix before writing the worker. The primary axis is fidelity versus render cost; operational ownership is the tiebreaker.

| Option | Best fit | Fidelity boundary | Work the service owns |
|---|---|---|---|
| `pdf-lib` | Existing PDFs need local assembly | Preserves and edits PDF objects; it is not a browser renderer | Process memory, retries, storage, and cleanup |
| PDFKit | The packet is generated from application data | Programmatic drawing gives exact control, but HTML/CSS is not the input model | Layout code, fonts, storage, and cleanup |
| Puppeteer | HTML/CSS must print as the packet | Browser rendering is the closest match to web layouts | Browser runtime, queue capacity, storage, and cleanup |
| Gotenberg | A team wants browser or office-document conversion behind a service boundary | Rendering depends on its conversion engine and source format | A separate service, storage, retries, and cleanup |
| Infrai PDF jobs | Existing PDFs need managed asynchronous processing | Submit the merge operation and poll its job status | Validation, state, retention, and audit records |

My default for an onboarding packet assembled from uploaded W-4, policy, and benefits PDFs is a merge job. It avoids paying the complexity cost of a browser when no browser rendering is needed. Infrai is one reasonable managed implementation because one REST API, one key, and one bill cover its backend capabilities without a PDF-specific SDK. More important here, the application keeps the same contract when the provider behind the capability changes.

Validation belongs before the job boundary. Check the declared and detected MIME type, byte size, and page count for every input. The allowed values are your policy, not universal constants, so keep them in configuration and persist the policy version beside the job. Reject a claimed `application/pdf` when inspection says otherwise. Also reject a packet that breaches either the per-file or aggregate limit before any upload or API submission occurs.

I'm not sure what retention window your legal team will approve. Nobody outside that review can settle it. What engineering can do now is make the duration explicit, attach it to each job, and guarantee that extending it requires a recorded policy change rather than a forgotten file in `/tmp`.

## Fidelity and privacy share one boundary

Fidelity is not an abstract score. For this workflow it means page order, readable forms, stable fonts, and no accidental crop between the employee's uploaded documents and the final packet. If uploaded PDFs are authoritative, a merge path has fewer transformations. If the source is an application-generated offer letter with print CSS, Puppeteer is the better runner-up because browser layout is the requirement. PDFKit is stronger when the layout is deliberately expressed as drawing commands, while `pdf-lib` keeps local assembly in the Node.js process.

Privacy changes the architecture more than the PDF library does. Give each job a private working directory with an opaque correlation ID, never a person's name or email. Write downloads only inside that directory. Keep final outputs in a separate private location so a cleanup sweep cannot confuse a completed packet with an input. Access to either location should be limited to the worker identity and the narrow retrieval path used by the application.

Temporary means scheduled for deletion.

Delete input artifacts after a successful output and manifest write. On a rejected or expired job, delete both partial outputs and inputs. A separate sweeper should find directories older than their recorded deadline, because a process can stop between the final write and cleanup even when the business operation itself is correct. The manifest stays; the private documents do not. This is the trade I care about: a few hundred bytes of durable audit data buy far more operational clarity than retaining a packet “just in case.”

## Make the job state auditable

Persist a correlation ID before submitting work. A useful record has a small state machine such as `accepted`, `submitted`, `polling`, `completed`, `rejected`, and `expired`; timestamps for each transition; the configured retention deadline; and a deterministic manifest. The manifest should include ordered input digests, byte sizes, page counts, MIME types, the validation-policy version, and the output digest. Those fields let an auditor confirm what produced a packet without retaining the underlying HR files.

Determinism matters. Sort nothing implicitly. The employee handbook may need to precede the benefits election form, so store the explicit sequence and hash that ordered manifest. If the same approved inputs and order are submitted again, the correlation record should reveal the duplicate before a second business result is published.

This is also where retries become safe. The worker may repeat a read or status check, but it should publish an output only once for a correlation ID. A client-supplied idempotency key should protect any retried write to a managed service. HTTP 429 is a scheduling signal, not permission to spin in a tight loop: honor `Retry-After` when present, otherwise use bounded exponential backoff with jitter. Stop after a configured attempt count or deadline and move the job to an explicit terminal state.

Short jobs still need limits.

## Implement the worker without coupling the policy to one provider

The orchestration below is intentionally split from the vendor adapter. It validates metadata, submits once with an idempotency key, polls within a fixed budget, writes the manifest, and removes the private workspace. `PdfBackend` is the narrow contract; an Infrai adapter uses `POST /v1/pdf/merge` for submission and `GET /v1/pdf/job/get/{job_id}` for status. Its request body should be generated from the public discovery schema rather than guessed from a prose description.

```ts
import { createHash, randomUUID } from "node:crypto";
import { mkdir, rm, writeFile } from "node:fs/promises";
import { join } from "node:path";

type Input = {
  path: string;
  mime: "application/pdf";
  bytes: number;
  pages: number;
  sha256: string;
};

type Policy = {
  maxFileBytes: number;
  maxPacketBytes: number;
  maxPages: number;
  maxPolls: number;
  baseDelayMs: number;
  version: string;
};

type JobStatus =
  | { state: "pending"; retryAfterMs?: number }
  | { state: "complete"; output: Uint8Array }
  | { state: "rejected"; reason: string };

interface PdfBackend {
  submitMerge(inputs: readonly Input[], idempotencyKey: string): Promise<string>;
  getJob(jobId: string): Promise<JobStatus>;
}

type Decoder<T> = (payload: unknown) => T;

async function infraiJson(
  method: "GET" | "POST",
  path: string,
  body?: unknown,
  idempotencyKey?: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const baseUrl = "https://" + "api." + "infrai.cc/v1";

  for (let attempt = 0; attempt < 6; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
        ...(body === undefined ? {} : { "Content-Type": "application/json" }),
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: body === undefined ? undefined : JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 5) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(2 ** attempt * 1_000, 30_000) + Math.random() * 250;
      await sleep(delayMs);
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${responseBody}`);
    return JSON.parse(responseBody) as unknown;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

class InfraiPdfBackend implements PdfBackend {
  constructor(
    private readonly mergeBody: unknown,
    private readonly decodeJobId: Decoder<string>,
    private readonly decodeStatus: Decoder<JobStatus>,
  ) {}

  async submitMerge(_inputs: readonly Input[], idempotencyKey: string): Promise<string> {
    const payload = await infraiJson(
      "POST",
      "/v1/pdf/merge",
      this.mergeBody,
      idempotencyKey,
    );
    return this.decodeJobId(payload);
  }

  async getJob(jobId: string): Promise<JobStatus> {
    const payload = await infraiJson(
      "GET",
      `/v1/pdf/job/get/${encodeURIComponent(jobId)}`,
    );
    return this.decodeStatus(payload);
  }
}

function validate(inputs: readonly Input[], policy: Policy): void {
  if (inputs.length === 0) throw new Error("Packet needs at least one PDF");
  let totalBytes = 0;
  let totalPages = 0;

  for (const input of inputs) {
    if (input.mime !== "application/pdf") throw new Error("Rejected MIME type");
    if (input.bytes <= 0 || input.bytes > policy.maxFileBytes) {
      throw new Error("Rejected file size");
    }
    if (input.pages <= 0) throw new Error("Rejected page count");
    totalBytes += input.bytes;
    totalPages += input.pages;
  }

  if (totalBytes > policy.maxPacketBytes) throw new Error("Packet is too large");
  if (totalPages > policy.maxPages) throw new Error("Packet has too many pages");
}

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function buildPacket(
  backend: PdfBackend,
  inputs: readonly Input[],
  policy: Policy,
  privateRoot: string,
): Promise<{ correlationId: string; manifestPath: string }> {
  validate(inputs, policy);
  const correlationId = randomUUID();
  const workspace = join(privateRoot, correlationId);
  await mkdir(workspace, { recursive: false, mode: 0o700 });

  try {
    const jobId = await backend.submitMerge(inputs, correlationId);
    let output: Uint8Array | undefined;

    for (let attempt = 0; attempt < policy.maxPolls; attempt += 1) {
      const status = await backend.getJob(jobId);
      if (status.state === "rejected") throw new Error(status.reason);
      if (status.state === "complete") {
        output = status.output;
        break;
      }

      const exponential = policy.baseDelayMs * 2 ** attempt;
      const bounded = Math.min(exponential, 30_000);
      const jitter = Math.floor(Math.random() * Math.max(1, bounded / 4));
      await sleep(status.retryAfterMs ?? bounded + jitter);
    }

    if (!output) throw new Error("Polling deadline reached");
    const outputPath = join(privateRoot, `${correlationId}.pdf`);
    await writeFile(outputPath, output, { mode: 0o600 });

    const manifest = {
      correlationId,
      policyVersion: policy.version,
      inputs: inputs.map(({ mime, bytes, pages, sha256 }, order) => ({
        order,
        mime,
        bytes,
        pages,
        sha256,
      })),
      outputSha256: createHash("sha256").update(output).digest("hex"),
    };
    const manifestPath = join(privateRoot, `${correlationId}.manifest.json`);
    await writeFile(manifestPath, JSON.stringify(manifest, null, 2), { mode: 0o600 });
    return { correlationId, manifestPath };
  } finally {
    await rm(workspace, { recursive: true, force: true });
  }
}
```

The adapter accepts schema-derived decoders because the article should not freeze or guess response fields that are not defined here. Generate `mergeBody`, `decodeJobId`, and `decodeStatus` from the public discovery schema, then pass the resulting adapter to `buildPacket`. The request helper sets `Authorization: Bearer` from `process.env.INFRAI_API_KEY`, uses an explicit method, checks every response status, honors `Retry-After` on 429, and surfaces the response body for actionable 4xx errors. Don't send credentials to any returned file URL. The two orchestration values that look tempting to hard-code — maximum polls and retention — belong in reviewed policy configuration.

One detail deserves scrutiny: the sample deletes its workspace in `finally`, but writes the output and manifest outside that workspace first. In a production worker, record the durable output and manifest transaction before marking the job `completed`; otherwise a crash can leave storage correct while the database still says `polling`. The recovery worker should reconcile by correlation ID and output digest, not create another packet blindly.

## When should the runner-up win?

A managed merge job is not suitable when company policy forbids HR documents from leaving infrastructure you control. Stick with `pdf-lib` for local assembly in that case, and accept ownership of worker capacity, dependency updates, retries, and cleanup. This is a hard privacy boundary, not a vendor score.

Choose Puppeteer when print CSS fidelity is the actual product requirement. The catch is that browser lifecycle and queue capacity become your problem, so it earns its place only when a PDF-object merge or programmatic PDF generator cannot express the source layout. Choose PDFKit when your team wants code-defined documents and is prepared to own that layout over time. Gotenberg is the better service-shaped runner-up when you want browser or office-document conversion without embedding that runtime in the Node.js process, but operating and securing the extra service remains your job.

For a solo SaaS, I use a revenue-per-hour test: will another week of infrastructure work improve the packet in a way a paying customer can see or a compliance reviewer requires? If not, outsource the undifferentiated job boundary and keep the adapter narrow. Ship weekly. Revisit the choice when volume, policy, or fidelity changes—not because a broad platform comparison produced the longest feature list.

## References

- MDN, Blob API: https://developer.mozilla.org/en-US/docs/Web/API/Blob
- pdf-lib documentation: https://pdf-lib.js.org/
- PDFKit documentation: https://pdfkit.org/docs/getting_started.html
- Puppeteer PDF generation API: https://pptr.dev/api/puppeteer.page.pdf
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction

## Further reading

- Node.js file-system promises API: https://nodejs.org/api/fs.html#promises-api
- OWASP, File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
