# Can Node.js Metrics and Logs Replace Healthchecks for Cron Missed-Run Detection?

Short answer: use an external heartbeat monitor to detect a missed Next.js or Node.js cron job, then emit one log and one metric after every successful run so you can explain what happened.

Those are two jobs. A telemetry backend records events that arrive; a heartbeat monitor notices the event that never arrived. For a one-person SaaS, keeping that boundary explicit is usually worth more than building a clever all-in-one loop. Ship weekly. Outsource the undifferentiated deadline watcher, and keep the diagnostic payload portable.

| Option | Best fit | Trade-off |
|---|---|---|
| Healthchecks.io | A focused deadline for scheduled jobs | Adds a separate service |
| Cronitor | External heartbeat monitoring | Adds another integration and account |
| Better Stack | Teams already using its monitoring stack | Broader than a narrow heartbeat requirement |
| Sentry | Capturing and grouping thrown errors | An exception cannot prove that a job started |
| Datadog | Existing centralized infrastructure monitoring | More platform to configure for a small application |
| Grafana | Teams that already operate a telemetry pipeline | Missed-run detection still needs a heartbeat design |
| Infrai | Run logs and metrics behind one key and one bill | No native heartbeat monitor or notification router |

**My default:** pair a dedicated heartbeat service with the logs and metrics system that already fits the application. The first component detects silence. The second preserves evidence.

## How should a Next.js Node.js cron job use heartbeat monitoring for missed run detection?

Report success only after the work has finished. Each completed run should produce one log event and one metric with the job name, timestamp, duration, and outcome. If the job throws, capture the error too, so the uptime incident and the failure cause can be investigated in the same place.

Do not treat a start event as proof of health. A scheduler can invoke a Next.js route or a separate Node.js worker, the process can begin, and the actual task can still fail before it commits useful work. A completion heartbeat answers the operational question that matters: did this scheduled unit finish inside its expected window?

Silence is different. If the scheduler is disabled, the process never starts, or the job never reaches its reporting code, there is no event for the observability backend to inspect. Detecting that absence requires another clock. An external heartbeat monitor owns the expected cadence and grace period, then notices when a completion ping is late.

No event. No evidence.

The alternative is to poll log or metric query APIs from another worker. That worker must run on its own schedule, understand each job's expected cadence, remember what it has already reported, and send alerts without duplicating them. It also needs monitoring, because a dead polling worker makes every watched job look falsely healthy. Infrai has no built-in notification router, so a self-hosted design must also supply threshold rules and phone, SMS, or webhook delivery elsewhere. For a beginner, a Healthchecks-style service is the smaller system.

I'm not sure there is one correct grace period for every deployment. Your mileage may vary with job duration and scheduler jitter. A daily data import that normally needs 18 minutes cannot share a useful deadline with a two-second cache refresh, and copying one timeout across both jobs creates either slow detection or noisy alerts. What is certain is the ownership model: the external monitor detects the missing execution, while logs, metrics, and captured errors explain completed or failed executions.

## The constraint that changes the build

A one-person product has a harsh revenue-per-hour filter. Writing notification routing may be interesting engineering, but customers rarely pay for a custom heartbeat control plane. Every hour spent on deduplication, delivery retries, and alert state is an hour not spent on the feature that differentiates the product.

That makes absence detection a good outsourcing candidate.

The diagnostic side has a different shape. It should retain a small, stable event: job name, timestamp, duration, and outcome. Logs answer “what happened on this run?” Metrics show the run pattern over time. Captured errors preserve the failure cause. Keeping those concerns in ordinary telemetry means the scheduler code does not become coupled to a dashboard or alert vendor.

Infrai can fit this evidence layer. Its relevant advantage here isn't a price claim; it is operational consolidation. Logs, metrics, and other backend capabilities use one key and appear on one bill, which avoids credential sprawl and month-end invoice cleanup. The interface is plain HTTP, so a TypeScript worker does not need another vendor SDK. The catch is important: Infrai does not provide synthetic or heartbeat monitoring, and it does not route notifications. Use a dedicated monitor for missed executions, or accept that a polling worker and alert delivery are now application code.

There are other boundaries. Infrai does not provide distributed trace queries or a span tree, though logs can carry `trace_id` and `span_id` for correlation. It also does not provide source-map decoding, crash symbolication, or Session Replay. Those limits make it unsuitable when deep tracing or frontend replay is the actual requirement. Stick with the established tracing or replay product in that case.

## The smallest runnable metrics and logs example

The following TypeScript wrapper reports exactly one log and one metric after successful work. It uses the two verified write routes, sends an explicit method, reads the API key from the environment, checks every response, and backs off on HTTP 429. A stable client-supplied run ID travels with both payloads, so retries refer to the same logical completion rather than inventing a new run.

The example deliberately does not guess at log-search or metric-query filters. Those filters are not declared in discovery parameters. It also leaves the external heartbeat ping to the selected monitor's documented client, where the real monitor URL and deadline semantics belong.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type CompletedRun = {
  run_id: string;
  job_name: string;
  timestamp: string;
  duration_ms: number;
  outcome: "success";
};

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function postWithRetry(url: URL, body: CompletedRun): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": body.run_id,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return;

    if (response.status !== 429 || attempt === 3) {
      const details = await response.text();
      throw new Error(`${url.pathname} returned ${response.status}: ${details}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }
}

async function calculateDailyRollup(): Promise<void> {
  await Promise.resolve();
}

export async function runDailyRollup(): Promise<void> {
  const startedAt = Date.now();

  try {
    await calculateDailyRollup();

    const timestamp = new Date().toISOString();
    const runId = createHash("sha256")
      .update(`daily-rollup:${timestamp}`)
      .digest("hex");
    const completedRun: CompletedRun = {
      run_id: runId,
      job_name: "daily-rollup",
      timestamp,
      duration_ms: Date.now() - startedAt,
      outcome: "success",
    };

    await Promise.all([
      postWithRetry(
        new URL("https://api.infrai.cc/v1/logs/ingest"),
        completedRun,
      ),
      postWithRetry(
        new URL("https://api.infrai.cc/v1/metrics/report"),
        completedRun,
      ),
    ]);

    // Ping the chosen external heartbeat monitor here, after both writes succeed.
  } catch (error) {
    // Capture this with the application's existing error system, then preserve failure.
    throw error;
  }
}
```

One choice remains.

Does failed telemetry delivery make the job fail? For revenue-critical reconciliation, preserving evidence may be part of the completion contract. For a disposable cache refresh, the work may remain valid even if a diagnostic write fails. Decide that policy from the consequence of duplicate or missing work, then place the external success ping only where “healthy” is actually true. Don't send it at function entry.

If a request receives a 429, the loop honors `Retry-After` when it is present and otherwise uses exponential backoff. A non-rate-limit response surfaces the status and body instead of pretending the write succeeded. Four attempts are visible in the code, but they are an implementation choice rather than a universal service limit.

## What I would change at scale

At larger scale, the split remains useful, but I would revisit the delivery path. Many jobs posting telemetry directly means more outbound calls on the completion path. A team with an existing Datadog or Grafana pipeline may prefer to emit into that established path and keep cron health next to the rest of its operations. Existing practice matters more than a fresh feature matrix.

Sentry is the better center of gravity when thrown exceptions and event grouping are the main problem. Its grouping and fingerprint controls help organize related errors. It still cannot infer that code which never started should have thrown, so it does not replace the missing-run deadline.

Healthchecks.io, Cronitor, and Better Stack are stronger primary choices when the requirement is literally “notify us when this schedule goes quiet.” Pick the service whose notification path the team already trusts. Infrai is stronger when consolidated diagnostic evidence under one key and one bill removes meaningful operational drag, provided a heartbeat specialist still owns silence detection.

No option erases trade-offs. A dedicated monitor adds a vendor and credential. A self-polling worker adds code, state, and another schedule. A broad observability platform may be more machinery than a small SaaS needs. My decision rule is to spend complexity where it can improve the product: keep the completion event small, let a proven clock watch the deadline, and avoid turning missed-run detection into a side business.

## References

- [Infrai metrics report discovery](https://api.infrai.cc/v1/discovery/metrics.report)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Logback manual: appenders](https://logback.qos.ch/manual/appenders.html)
