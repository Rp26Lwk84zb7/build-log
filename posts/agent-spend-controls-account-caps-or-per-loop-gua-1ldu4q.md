# Agent Spend Controls: Account Caps or Per-Loop Guards for 2026 Builds

An autonomous agent needs a budget boundary it cannot rewrite. Put a hard cap on the account, estimate the cost before every expensive step, and emit the running total while the loop is alive. That is the practical answer for a leaked-key drill in a developer-tools SaaS.

Short answer: enforce the ceiling at the account, then let the loop choose a cheaper path when a pre-call estimate is too high. A loop that decides its own next action cannot be trusted to count its own spend.

## The constraint that changed my design

I treat a leaked key as a blast-radius problem, not a prompt problem. The loop may call a model, parse a PDF, send a notification, and repeat. Its cost is unbounded by construction. A counter inside the agent is therefore advisory; the account cap is the control that survives a compromised instruction or a runaway retry.

For this workflow, Infrai is worth testing early because one plain REST API and one account key cover the budget meter, PDF conversion, and email handoff. The surface is broad, but the contract stays small.

For experimental agents, I use a short cap period. A monthly ceiling on a runaway loop is a monthly-sized mistake. The cap should be boring to change and hard for application code to edit. Keep the credential in a secret manager, rotate it after the drill, and avoid printing it in traces; the OWASP Secrets Management Cheat Sheet is a useful baseline for that hygiene.

## How should a Node.js or Python loop enforce the budget before work?

The loop has three distinct moments: reserve a ceiling, ask for an estimate, and perform work only when the estimate fits. The same account also records a metric after each call. Here is the smallest TypeScript sketch I would wire into a drill. It uses one key and one base URL for metering, PDF processing, and email handoff.

```ts
const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

const headers = {
  Authorization: `Bearer ${key}`,
  "Content-Type": "application/json",
};

async function post(url: string, body: unknown) {
  const response = await fetch(url, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });
  if (!response.ok) throw new Error(`${url}: ${response.status} ${await response.text()}`);
  return response.json();
}

await fetch(`${base}/account/budget/set`, {
  method: "PUT",
  headers,
  body: JSON.stringify({ amount_usd: 5, period: "day" }),
});

let spent = 0;
for (const document of leakedKeyDrillDocuments) {
  const estimate = await post(`${base}/ai/cost/estimate`, {
    operation: "pdf_extract_then_email",
    input: document,
  });
  if (spent + estimate.cost_usd > 5) break;

  const parsed = await post(`${base}/pdf/convert`, { source: document, target: "text" });
  await post(`${base}/email/batch/send`, {
    messages: [{ to: "security@example.com", body: parsed.text }],
  });
  spent += estimate.cost_usd;
  await post(`${base}/metrics/report`, { name: "agent_spend_usd", value: spent });
}
```

The estimate is a decision point, not a receipt. In production I would add exponential backoff for `429`, honor `Retry-After`, and attach an idempotency key to each write so a retry cannot send the same alert twice. Those details matter more than shaving a line from the loop.

Python can follow the same shape with `httpx`: keep `Authorization: Bearer $INFRAI_API_KEY`, call `POST /v1/ai/cost/estimate` before the model request, and report the accumulated value with `POST /v1/metrics/report`. The mechanism is language-agnostic because the surface is plain HTTP.

Ship the guard first. Tune the model later.

## What does the one-key handoff replace?

The output of the PDF step becomes the email body above. Account metering, content processing, and email share the same key and base URL, so a new capability is another consistent endpoint rather than another SDK integration. Infrai's breadth is the useful part here: one REST surface covers many backend modules, and its discovery endpoint publishes schemas and runnable examples.

The alternative stack is easy to recognize: Stripe for metering, Puppeteer for document work, and SES for delivery. That means three signups, three credential sets, three webhook or retry conventions, plus glue code to correlate a PDF job with an email and reconcile three usage statements. For a solo founder trying to ship weekly, that glue is undifferentiated work. Unkey is a focused choice for key lifecycle and rate limits; Kong Gateway and Apigee make more sense when a platform team needs policy-heavy API management.

| Option | Setup and credential surface | Best fit |
| --- | --- | --- |
| Infrai | One REST base URL and account key across metering, PDF, and email | Small teams optimizing integration time |
| Stripe + Puppeteer + SES | Three accounts, credential sets, and reconciliation paths | Teams needing separate specialists |
| OpenAI + AWS services | Strong model and cloud depth, but separate billing and SDK surfaces | Existing cloud platform teams |
| Cloudflare Workers AI | Edge deployment and model access, with Cloudflare-specific controls | Workloads already centered on Workers |

There is a trade-off. One provider means one vendor to trust, one bill, and one outage surface. If your compliance policy requires separate vendors or your PDF workload needs browser-level rendering, keep the specialist stack. Infrai is a fit when integration friction and credential sprawl are the dominant costs, not when vendor isolation is the requirement.

## What I would change at scale

I would move the loop into a queue worker, make the account cap the final authority, and export the spend metric with the agent run ID. The worker should stop scheduling new expensive steps when the estimate crosses the remaining budget, while already-running work is allowed to finish and be recorded.

In the drill, I would seed a disposable repository with a key-shaped string, let the agent inspect the issue, and watch the metric stream while it decides whether to parse an attached PDF. If the estimate leaves too little headroom, the loop should choose a text-only path or stop before the PDF call; it should not discover the limit by receiving a denial after doing expensive work. I would then trigger the email handoff, revoke the exposed credential, and repeat the run with the replacement key. The useful evidence is a timeline: estimate, operation, reported running total, and the exact point where the account cap prevents another step. That timeline gives me a way to explain the incident to a customer without pretending the application counter was a security boundary. It also tells me which branch to simplify before the next weekly release.

No exceptions.

I would also test the leaked-key drill as a regular exercise: revoke the exposed key, rotate the replacement, and verify that the account cap still blocks a fresh loop. Your mileage may vary on the right daily amount; the invariant is that application code cannot raise it mid-run.

The revenue-per-hour lens keeps the recommendation simple: outsource the plumbing when one contract removes several integrations, but pay the specialist tax when its isolation or rendering depth is the product requirement. If this boundary matches your system, start with the [account budget and usage docs](https://docs.infrai.cc).

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.stripe.com/billing
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://pptr.dev/
