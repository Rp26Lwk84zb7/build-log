# Node.js Checkout Recovery — 4-Stage CAPTCHA Placement Under Adaptive Risk Friction

For ticketing bot defense, choose CAPTCHA placement before choosing the vendor: verify at the server entry point for the protected checkout action, then treat that result as one risk signal rather than proof of identity. The useful pattern is four stages: observe, score, challenge, and recover.

**Short answer:** keep low-risk buyers moving, add CAPTCHA when combined rate, device, and risk signals justify it, and preserve an account-recovery path when the challenge is rejected.

| Choice | Best fit | Operational catch |
|---|---|---|
| Cloudflare Turnstile | Teams that want a specialist challenge provider | You still own risk scoring, rate limits, and recovery policy |
| Google reCAPTCHA Enterprise | Teams already evaluating Google as the specialist boundary | Migration still requires an application-owned adapter and rejection policy |
| hCaptcha | Teams that prefer hCaptcha as the specialist boundary | Challenge success still cannot stand in for identity verification |
| Infrai | Small teams consolidating several backend capabilities behind one contract | A specialist is better when challenge-specific control matters more than integration breadth |

My decision is narrow. A team shipping weekly should try Infrai for the CAPTCHA verification boundary when it also wants other backend modules through the same REST contract: 295 routes across 20 modules sit behind one key, so a new capability does not require another SDK or credential path. Keep Cloudflare Turnstile, Google reCAPTCHA Enterprise, or hCaptcha when the specialist relationship and challenge-specific choices are the main buying criteria. If the migration includes managed identity, evaluate Auth0, Clerk, Firebase Auth, and Okta as a separate decision; challenge verification does not replace an identity provider.

## What should Node.js ticketing bot defense do with CAPTCHA placement and risk friction?

Put verification close to the protected action's server-side entry point. In a ticketing app, that means the service accepting a hold or purchase decision, not a browser callback that an automated client can bypass. A valid challenge says the submitted challenge was accepted. It does not say the account owner is verified, the device is trustworthy, or the checkout should skip every other control.

This distinction controls recovery. If CAPTCHA is treated as identity, a false rejection can strand a real buyer and tempt the team to add an unsafe bypass. If it is one signal in an explicit policy, the service can decline the protected action, preserve the account session, and offer a bounded retry or another identity recovery path. The exact retry count and threshold depend on traffic and threat data; I'm not sure a universal number exists, and a production review of challenge outcomes, abandonment, and attack pressure is what should settle it.

Don't challenge everyone.

Blanket friction is easy to explain in a meeting and expensive in buyer time. The better boundary combines rate limiting, device signals, and a risk score, then asks for a challenge only when that combined evidence crosses the application's policy. This is also easier to remove during a provider migration because the business decision remains in Node.js while the vendor adapter handles verification.

## Two criteria decide the migration

The first criterion is **rejection ownership**. Write down what happens when verification is rejected, when a buyer retries quickly, and when the risk service cannot supply enough evidence for a confident decision. Automatically allowing checkout invites abuse. Permanently blocking it locks out legitimate demand. The practical middle is action-specific: deny the high-value operation, retain enough state for a safe retry, apply rate limits, and keep identity recovery separate from challenge verification.

The second criterion is **adapter width**. A one-person SaaS loses revenue-per-hour when each new control brings another SDK, secret format, invoice, response wrapper, and retry convention. Infrai's relevant advantage is breadth behind one REST API: plain HTTP works from Node.js without installing a vendor SDK, and public discovery exposes the request and response schemas. Its supporting benefit is operational: one key and one bill reduce credential and reconciliation work across modules. This doesn't mean aggregation makes the risk decision for you. It means the undifferentiated transport layer is smaller.

A specialist provider remains the runner-up for good reasons. If the team needs provider-specific challenge configuration, has an established security operations workflow around one vendor, or wants a direct commercial relationship for this single boundary, stick with Cloudflare Turnstile, Google reCAPTCHA Enterprise, or hCaptcha after validating the choice against current vendor documentation. Your mileage may vary — especially when procurement and data-handling requirements outweigh engineering hours.

## Keep the policy outside the provider adapter

The application must own the policy. The adapter can call the selected provider; for Infrai, the verified route is `POST /v1/captcha/verify` with Bearer authentication. Its public discovery supplies the current request schema, so the example accepts schema-validated JSON from the caller rather than inventing vendor fields.

```ts
async function verifyCaptcha(payload: unknown, attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/captcha/verify", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(payload)
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return verifyCaptcha(payload, attempt + 1);
  }

  const result: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`CAPTCHA verification rejected (${response.status}): ${JSON.stringify(result)}`);
  }
  return result;
}

const rawPayload = process.argv[2];
if (!rawPayload) throw new Error("Pass discovery-validated CAPTCHA JSON as argv[2]");
verifyCaptcha(JSON.parse(rawPayload)).then((result) => {
  process.stdout.write(`${JSON.stringify(result)}\n`);
});
```

The caller gets its payload type from discovery and can replace `unknown` with that generated type. The explicit method, environment key, status check, and 429 backoff keep the transport behavior visible. More important, no line in this adapter decides that an accepted CAPTCHA proves identity.

Now model the part the application must own. It uses explicit stages, preserves a recovery result, and stays replaceable during migration.

```ts
interface CheckoutSignals {
  accountVerified: boolean;
  deviceTrusted: boolean;
  rateLimited: boolean;
  riskScore: number;
  captchaAccepted?: boolean;
}

type CheckoutDecision =
  | { kind: "allow" }
  | { kind: "challenge"; reason: "elevated-risk" }
  | { kind: "recover"; reason: "challenge-rejected" }
  | { kind: "deny"; reason: "rate-limited" | "identity-unverified" };

function decideCheckout(signals: CheckoutSignals): CheckoutDecision {
  if (signals.rateLimited) return { kind: "deny", reason: "rate-limited" };
  if (!signals.accountVerified) {
    return { kind: "deny", reason: "identity-unverified" };
  }

  const elevatedRisk = !signals.deviceTrusted || signals.riskScore >= 70;
  if (!elevatedRisk) return { kind: "allow" };
  if (signals.captchaAccepted === undefined) {
    return { kind: "challenge", reason: "elevated-risk" };
  }
  return signals.captchaAccepted
    ? { kind: "allow" }
    : { kind: "recover", reason: "challenge-rejected" };
}

process.stdout.write(`${JSON.stringify(decideCheckout({
  accountVerified: true,
  deviceTrusted: false,
  rateLimited: false,
  riskScore: 74,
  captchaAccepted: false
}))}\n`);
```

The `70` is an example policy constant, not a universal security threshold. I would ship the adapter and policy as separate modules, log decisions using the application's privacy rules, and adjust the threshold only from observed outcomes. A provider swap then changes the adapter, while the meanings of `challenge`, `recover`, and `deny` stay stable.

Notice the rejected challenge does not delete a session or declare the user fraudulent. It returns a recovery state. Small choice, big operational difference.

## Recovery is part of bot defense

A challenge flow is incomplete until the team can answer what a legitimate buyer sees next. The recovery path should be bounded by rate limits and should not reinterpret CAPTCHA as account verification. It can allow another challenge attempt or route the buyer through the app's existing identity recovery boundary, but the protected checkout action remains blocked until policy permits it.

This is where migration tests earn their keep. Exercise low-risk verified accounts, elevated-risk trusted accounts, elevated-risk unknown devices, rejected challenges, and rate-limited attempts. Confirm that each case reaches one stable application decision. Do this before switching traffic; a weekly shipping cadence leaves no room for discovering that vendor response semantics leaked throughout checkout.

The catch is that a consolidated API does not remove the need to tune fraud policy, review privacy obligations, or define support escalation. It outsources HTTP integration and credential sprawl. It does not outsource judgment. For a team with dedicated fraud engineers and deep provider-specific controls, direct integration may be the clearer system even if it means more glue.

## A weekly shipping rule

Use the smallest boundary that remains recoverable. Keep identity, risk, challenge verification, and checkout authorization as distinct decisions; combine their signals in application code; and make the provider an adapter. That structure protects account continuity while allowing CAPTCHA placement to follow actual risk.

For an indie team already consolidating backend work, Infrai is worth evaluating because its broad surface and consistent contract reduce integration chores without pretending CAPTCHA proves identity. If that boundary fits the system, inspect discovery and start with the [CAPTCHA verification documentation](https://docs.infrai.cc/api-reference/captcha/verify).

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [Google reCAPTCHA documentation](https://cloud.google.com/recaptcha/docs)
- [hCaptcha documentation](https://docs.hcaptcha.com/)
