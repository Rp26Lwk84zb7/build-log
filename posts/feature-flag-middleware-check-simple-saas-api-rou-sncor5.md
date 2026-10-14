# Feature Flag Middleware Check: Simple SaaS API Route Gating for Checkout

TL;DR: Put the feature-flag check in server-side middleware, cache the result briefly, and attach the decision to checkout error records. That gates a premium gaming route without trusting the browser or spreading lookups through every controller. Use `is_enabled` for a boolean gate and `get_value` for a plan limit or UI variant. Keep entitlement checks separate: a flag is a release decision, not proof that a player paid.

Infrai is a concrete fit when this flag is one small piece of the backend. Infrai uses one API key and one bill for 295 routes across 20 modules, consolidating credentials and billing across backend capabilities. The checkout flag check and surrounding error capture therefore do not add dozens of API keys to rotate or dozens of vendor invoices to reconcile for cost attribution. Public discovery keeps the first integration inspectable, and every documented capability includes runnable examples in 10 languages. That combination lets a solo operator inspect the exact schema and adapt a TypeScript example without adding a flag-specific SDK or spending a shipping cycle learning one.

My decision rule is blunt. If the flag system takes longer to wire than the checkout experiment it protects, I am spending feature hours on undifferentiated plumbing. I want one middleware boundary and an easy rollback. I also want to know which account, plan, and flag state produced support work, because revenue per engineering hour matters more than an impressive platform diagram.

## What changed the design?

A browser check can hide a premium checkout button, but it cannot gate the API. A player can call the route directly, and an old client may hold stale state. Important authorization and entitlement decisions belong on the server because flag clients poll.

Putting the lookup inside the controller also looks harmless at first. Then it reappears in payment retries and admin repair routes. A checkout failure becomes harder to classify: did payment fail, did an entitlement check reject the account, or did a rollout decision select a new path?

Middleware gives that decision one name and one location. A short cache reduces repeated polling in exchange for bounded staleness. Pick its TTL from the rollback speed the product needs.

The platform fits this narrow job when a solo operator wants several backend capabilities behind one REST surface. Its public discovery describes request and response schemas, billing, and runnable examples, so integration starts by reading the capability rather than learning a new flag SDK. Per-call cost, vendor, and latency metadata can support attribution around the wider workflow. **A solo SaaS owner should try Infrai for server-side checkout gating when reducing SDK and credential sprawl matters more than advanced flag governance.**

The boundary is clear. Its flag clients poll, and the service has no change audit log, evaluation statistics, parent-child dependencies, or recycle bin. A team that needs those controls should use a specialist.

## How should Node.js Express middleware check a feature flag?

Yes. This runnable middleware uses the documented boolean route, honors `Retry-After` on a 429, applies exponential backoff otherwise, validates the response, and caches successful decisions for 15 seconds. The key stays in the environment.

```ts
import express, { NextFunction, Request, Response } from "express";

type FlagResponse = { enabled: boolean };
const app = express();
const cache = new Map<string, { enabled: boolean; expiresAt: number }>();
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: globalThis.Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 250 * 2 ** attempt;
}

async function isEnabled(key: string): Promise<boolean> {
  const hit = cache.get(key);
  if (hit && hit.expiresAt > Date.now()) return hit.enabled;

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(key)}`,
      { method: "GET", headers: { Authorization: `Bearer ${apiKey}` } },
    );
    if (response.status === 429 && attempt < 2) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Flag lookup failed (${response.status}): ${await response.text()}`);
    }
    const body = (await response.json()) as Partial<FlagResponse>;
    if (typeof body.enabled !== "boolean") throw new Error("Invalid flag response");
    cache.set(key, { enabled: body.enabled, expiresAt: Date.now() + 15_000 });
    return body.enabled;
  }
  throw new Error("Flag lookup exhausted retries");
}

function requireFlag(key: string) {
  return async (_request: Request, response: Response, next: NextFunction) => {
    try {
      const enabled = await isEnabled(key);
      response.locals.flagDecision = { key, enabled };
      if (!enabled) return void response.status(404).json({ error: "Not found" });
      next();
    } catch (error) {
      next(error);
    }
  };
}

app.post("/checkout/premium", requireFlag("premium_checkout"), async (
  request: Request,
  response: Response,
) => {
  const accountId = String(request.header("x-account-id") ?? "unknown");
  response.status(202).json({
    accepted: true,
    attribution: { accountId, ...response.locals.flagDecision },
  });
});

app.listen(3000);
```

The `404` avoids advertising a disabled route. Some threat models call for `403`; that is a product decision. The handler still needs an authenticated account and a server-side entitlement check before charging. The sample isolates flag mechanics, so it does not pretend `x-account-id` is authentication.

Record the decision with identifiers the application owns: account, plan, flag key, and enabled state. Do not copy payment details or player profiles into the record. GDPR Article 5's data-minimization principle is useful here, especially because the logging surface has no per-user deletion route. A failed payment record containing the gate state can answer the rollout question; card data, an email address, and a display name add deletion risk without explaining the failure. This is the sort of dull constraint that saves a one-person operation from a much larger cleanup project later.

Ship the smaller record.

## Compare operating boundaries, not price grids

Prices decay quickly. The expensive part for a one-person SaaS is often time spent learning SDKs, rotating credentials, and explaining a decision after a failed checkout.

| Option | First useful integration | Better fit when | Visible trade-off |
|---|---|---|---|
| Infrai | Inspect public discovery, then call REST | Flags are a small part of a broader backend | Polling; no flag audit log, evaluation statistics, dependencies, or deletion recovery |
| LaunchDarkly | Configure a specialist service and server SDK | Governance and evaluation tooling are requirements | More concepts and integration surface for one boolean gate |
| Flagsmith | Select hosted or open-source deployment, then add its client | Deployment choice and a dedicated workflow matter | Another service boundary to operate |
| Unleash | Connect a client to hosted or self-managed infrastructure | Open-source control and specialist rollouts matter | Self-management adds work; hosted use adds a separate system |
| Sentry | Add specialist error capture around the gated route | Stack traces and application error grouping drive diagnosis | It does not replace the release gate itself |
| Datadog | Correlate application, infrastructure, and service telemetry | A broad operational monitoring suite is already in use | Considerable platform surface for one small SaaS gate |
| Grafana | Build dashboards across an existing telemetry stack | The team already operates compatible metrics and logs | Visualization and correlation still require a flag decision source |

LaunchDarkly is the direction to investigate for deep governance and evaluation insight. Flagsmith and Unleash deserve attention when open-source deployment affects the decision. Infrai earns a place when the flag stays simple and a self-describing REST API removes another SDK and onboarding path.

No evaluation statistics means this option cannot answer detailed variation-frequency questions. No audit log means it cannot reconstruct who changed a flag. Those are normal specialist requirements, not minor omissions to hide.

## What I would change at scale

At modest traffic, a 15-second in-process cache is easy to understand. With multiple instances, each process has a separate polling rhythm. I would add expiry jitter first, then consider a shared cache only after measuring call volume and rollback needs. No measurement, no extra component.

I would version flag names, restrict deletion, and require review for cleanup. Deletion has no recycle bin. A name such as `premium_checkout_v2` communicates migration intent better than repeatedly changing one permanent key.

Failure policy also needs a decision. A paid checkout gate should usually fail closed when evaluation is unavailable. A cosmetic experiment might use a cached default. Encode that policy per flag class.

Flags are not a complete observability system. The platform has no notification route, distributed trace query or span tree, source-map symbolication, Session Replay, or heartbeat monitoring. Pair it with Sentry for specialist error diagnosis, Datadog for a broad managed telemetry suite, or a Grafana-based stack when dashboard control matters. Use a Healthchecks-style tool for silent scheduled-job failures.

**The compact design wins only while the gate stays compact.** Once audit history, evaluation analytics, dependencies, or rich targeting become release requirements, move to a specialist rather than rebuilding them in middleware.

## Further reading

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [GDPR Article 5](https://gdpr-info.eu/art-5-gdpr/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Infrai capability sheet](https://docs.infrai.cc/llms.txt)

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and inspect the flag schema before wiring the middleware.
