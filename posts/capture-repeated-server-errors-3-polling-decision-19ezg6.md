# Capture Repeated Server Errors — 3 Polling Decisions for Nightly Fintech Exceptions

Choose by who owns the next action: capture server exceptions centrally, poll repeated error groups, and keep enough structured evidence to attribute the investigation to a nightly fintech run.

| Choice | Useful role in this decision | Boundary to evaluate |
| --- | --- | --- |
| Sentry | Candidate for the exception investigation workflow | Evaluate the debugging context your application needs |
| Datadog | Candidate when the decision spans logs and operational monitoring | Evaluate the configuration and ongoing ownership involved |
| Grafana Loki | Candidate when searching structured logs is the main job | Evaluate who will maintain the surrounding alert workflow |
| Healthchecks | Complement for a nightly job that never reports completion | A missed run and a repeated exception need different signals |

TL;DR: Start with grouped exception capture and a recent-window threshold. For a one-person SaaS, the deciding metric is revenue per hour: how quickly can an alert identify the failed run, its owner, and the work needed to recover? Pick a specialist when its investigation workflow earns back the additional integration work.

## How should server exceptions trigger an alert on repeated errors?

A nightly settlement pipeline has two questions to answer. Did it finish, and did the same failure keep happening while it ran? Central exception capture answers part of the second question. A completion signal answers the first. Searching structured logs should connect both to the same run so the person investigating does not have to reconstruct identity from timestamps alone.

Count recent failures.

Consider a proposed rule of 3 occurrences of the same group within 10 minutes, checked every 2 minutes. These are starting settings for an example, not measured recommendations. The alert should identify the group and the run whose records need attention. A threshold on an all-time count would be misleading: yesterday's resolved work could keep today's poll above the threshold even when no new failure occurred. Define the window before choosing the count.

Capture an exception at the application boundary that owns it. In an Express application, that means forwarding the error into central capture from the error-handling path. For a background pipeline, capture it at the worker boundary, with the run context still available. Avoid capturing the same thrown exception at every layer; that would turn one failure into several apparent occurrences. Grouping reduces noise, but it does not repair duplicate instrumentation.

The polling worker should query unresolved groups and compare recent occurrences with its threshold, then record which group and window it already notified about. Keep that notification state across restarts. Otherwise, restarting the worker can repeat an interruption without adding any evidence. Notification delivery is a separate responsibility in this design; select Slack or email according to where the operator will act. A missed completion needs a heartbeat tool such as Healthchecks, because an exception collector cannot observe a job that never started.

## Attribute work to a run before attributing a bill

The first criterion is attribution quality. Design a small application-owned log vocabulary: run identifier, pipeline stage, error group reference, and a business-safe cost center. These are proposed fields for your own records, not a vendor request schema. A useful search can then distinguish a reconciliation failure from a later retry of the same input. Do not put account secrets or raw payment records into diagnostic context.

One invoice cannot do that work.

Infrai offers a single REST API with no SDK required, using one key and one bill across storage and observability. Its broad capability surface covers 295 routes across 20 modules; the useful part here is keeping the evidence read and error capture in one worker with a consistent interface. Public discovery supplies request and response schemas without a key, so the handoff can be checked before adding credentials. Per-call metadata includes cost_usd, vendor, latency_ms, cache_hit, and request_id; preserve it alongside the application-owned run identifier for attribution.

A request identifier and a pipeline identifier answer different questions. The first helps identify a particular call. The second must survive a chain of calls, retries, and exception capture. A provider's charge also does not measure the founder's time spent investigating. Keep those dimensions separate in any internal report, then decide whether better context is reducing the work between an alert and a repair. There is no measured savings claim here.

## Outsource the undifferentiated, own the handoff

The second criterion is maintenance ownership. Ship weekly. A system that requires a fresh dashboard investigation every time the pipeline changes competes directly with feature work, so the handoff between stored evidence and an error record deserves more attention than the length of a vendor's feature list.

Both operations use one REST API over plain HTTP, with no SDK to install. That keeps the storage read and exception write in the same small worker instead of adding two client-library lifecycles to its maintenance list.

A combined service means one vendor to trust, one bill, and one outage surface. Accept that concentration deliberately. The trade-off is fewer credentials versus a shared dependency. An independent storage provider and an independent investigation tool give you separate operating relationships, but you must connect their identifiers, credentials, and recovery procedures yourself.

The database-and-release alternative makes this concrete: Neon plus LaunchDarkly entails 2 vendor signups and 2 sets of credentials. PlanetScale plus LaunchDarkly has the same two-provider boundary. Your application would own the glue that associates a database change or snapshot with the flag controlling the new path, as well as the decision about what should happen if only one side changes. That comparison does not establish database-branch or snapshot support for the storage interface used below. The example is narrower: it carries storage context into exception capture using the same credential.

Keep the release decision explicit. An error threshold should produce an investigation signal; deciding to revert a data change or change a flag requires a separate policy about data compatibility. A count alone does not establish that reversing a release is safe.

## Carry storage evidence into an exception record

The following TypeScript program demonstrates the seam without guessing capture fields. It reads bucket metadata and substitutes the serialized response into a capture payload supplied by the operator. Use the service's current capture schema to prepare CAPTURE_TEMPLATE_JSON, placing the literal marker __STORAGE_RESULT__ inside a supported string field. The template must describe the server exception you are capturing. STORAGE_BUCKET identifies its evidence bucket. INFRAI_API_KEY supplies the credential. Set API_BASE_URL to the service's documented HTTPS API root including its version prefix. Run the file in a TypeScript-capable Node environment.

This is a manually invoked bridge for one captured exception; it doesn't run the polling scheduler. The read response is carried as opaque evidence. Parse billing metadata or select individual storage fields using the published response schema.

```ts
import { randomUUID } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

async function main(): Promise<void> {
  const key = required("INFRAI_API_KEY");
  const base = required("API_BASE_URL").replace(/\/$/, "");
  const bucket = encodeURIComponent(required("STORAGE_BUCKET"));
  const template = required("CAPTURE_TEMPLATE_JSON");
  const parsed: unknown = JSON.parse(template);
  if (!parsed || typeof parsed !== "object" || Array.isArray(parsed)) {
    throw new Error("Capture template must be a JSON object");
  }
  if (!template.includes("__STORAGE_RESULT__")) {
    throw new Error("Capture template needs the storage marker");
  }

  async function request(
    path: string,
    method: "GET" | "POST",
    body?: string,
    idempotencyKey?: string,
  ): Promise<unknown> {
    for (let attempt = 0; attempt < 5; attempt++) {
      const headers: Record<string, string> = {
        Authorization: `Bearer ${key}`,
      };
      if (body !== undefined) headers["Content-Type"] = "application/json";
      if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;
      const response = await fetch(`${base}${path}`, {
        method, headers, body, signal: AbortSignal.timeout(30_000),
      });
      const text = await response.text();
      if (response.status === 429 && attempt < 4) {
        const retryAfter = response.headers.get("Retry-After");
        let serverDelay = 0;
        if (retryAfter !== null) {
          const seconds = Number(retryAfter);
          serverDelay = Number.isFinite(seconds)
            ? Math.max(0, seconds * 1000)
            : Math.max(0, Date.parse(retryAfter) - Date.now());
        }
        const delay = Math.max(
          1000 * 2 ** attempt,
          Number.isFinite(serverDelay) ? serverDelay : 0,
        );
        let remaining = delay;
        while (remaining > 0) {
          const chunk = Math.min(remaining, 60_000);
          await new Promise<void>((resolve) => setTimeout(resolve, chunk));
          remaining -= chunk;
        }
        continue;
      }
      if (!response.ok) {
        throw new Error(`${method} ${path}: ${response.status} ${text}`);
      }
      return text ? JSON.parse(text) : null;
    }
    throw new Error("Retry budget exhausted");
  }

  const storage = await request(`/storage/bucket/get/${bucket}`, "GET");
  const evidence = JSON.stringify(storage);
  const payload = JSON.parse(template, (_name, value: unknown) =>
    typeof value === "string"
      ? value.replaceAll("__STORAGE_RESULT__", () => evidence)
      : value,
  );
  const captureId = randomUUID();
  await request("/errors/capture", "POST", JSON.stringify(payload), captureId);
  process.stdout.write("Exception captured with storage context.\n");
}

main().catch((error: unknown) => {
  process.stderr.write(`${String(error)}\n`);
  process.exitCode = 1;
});
```

The write reuses its idempotency key across rate-limit retries. If you turn this into a durable worker, persist that key with the exception before sending; launching the program again creates a new key. Review the storage response before including it in a diagnostic record, and narrow the evidence to the permitted fields for your fintech environment. Keep stored artifacts private or signed-only, and use presigned URLs for artifact access. Never forward the API authorization header to a presigned URL.

The next layer is the group poller described earlier. Its test case should include one old group with no new occurrences, one genuinely rising group, and a process restart after a notification. Those cases distinguish a recent-window alert from a repeated reminder about historical data. Keep notification delivery and its deduplication record visible when you evaluate the setup.

## When the specialist is the better choice

Sentry is the runner-up to evaluate when the expensive part is understanding an exception rather than consolidating backend credentials. Put a representative failure through its investigation workflow before committing. Datadog belongs in that evaluation when log investigation and broader operational monitoring need to be assessed together. Grafana Loki is worth evaluating when structured-log search is already the center of the workflow and someone can own the surrounding alerting configuration. These are selection boundaries, not benchmark rankings; no comparative latency or operating-cost measurement is available here.

For browser or mobile crash work, make source-map processing, symbolication, and replay explicit acceptance tests. Trace identifiers in logs can help correlate records, but they do not themselves provide a distributed span tree. For the nightly pipeline, a missed-run signal still needs its own completion check.

The choice should survive an ordinary week of shipping. Ask whether a notification names the affected run, whether its recent count can be explained, and whether the associated evidence makes cost attribution possible. If the answer requires several manual joins each morning, the next improvement is the handoff and context, before a more elaborate threshold.

## References

- https://docs.sentry.io/
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/
- https://healthchecks.io/docs/
- https://neon.com/docs/introduction
- https://planetscale.com/docs
- https://launchdarkly.com/docs/
- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/
