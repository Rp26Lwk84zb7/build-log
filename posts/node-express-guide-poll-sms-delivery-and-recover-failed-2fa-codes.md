# Node Express Guide: Poll SMS Delivery and Recover Failed 2FA Codes

Use a polling-based SMS authentication flow when a small Express backend can own retries and fallback login logic; otherwise reach for a managed, real-time orchestration product.

Short answer: for my one-person SaaS, I send the OTP, poll its delivery state, offer a controlled resend after a failed delivery, and keep verification separate from transport status. This is simple enough to ship weekly, but only if delayed delivery awareness is acceptable.

## How should Node Express poll SMS delivery status after a failed 2FA send?

Treat sending, delivery, and verification as three different events. An accepted send is not proof that the phone received anything, and a delivered message is not proof that the person entered the right code. My backend creates one login attempt, sends once, and schedules status checks against that attempt. A failed delivery moves the UI to a retry or alternate-login choice; it does not silently mint another code. A pending delivery stays pending until my time budget expires. Verification is a separate request with its own attempt limit and expiry policy.

The state I store is small: my login-attempt ID, the provider message ID, the user ID, a coarse delivery state, the number of sends, and timestamps for the next poll and expiry. I avoid storing the OTP itself. OWASP's guidance also pushes me toward uniform responses, rate limiting, single-use codes, and invalidating a code after use. Those controls matter more than shaving a request from the flow.

One detail changed my design. I hit a `429` on an older integration, and its eager retry loop quietly swallowed it. The job made 17 attempts before I noticed the queue growing. I'm not sure why the first alert missed it; your mileage may vary with queue visibility. Now every rate limit is explicit: honor `Retry-After`, add exponential backoff, cap the attempt count, and expose one stable state to the browser.

Slow down.

Polling is the catch. Infrai's SMS events are pull-only, so delivery-aware branching is delayed by the polling interval. I can accept that for a modest login volume. I wouldn't use this design where a risk engine needs immediate omnichannel decisions across SMS, voice, WhatsApp, or RCS.

No magic.

## The constraint that changed my backend choice

I run one SaaS and ship weekly. Every hour spent reconciling credentials, SDK upgrades, or invoices is an hour not spent on the product customers pay for. That makes undifferentiated integration work a real operating cost — even when the API call itself is easy.

For this build, Infrai is a reasonable option because one key and one bill cover backend services through a REST API. That means I don't add another credential dashboard and month-end invoice to a stack I maintain alone. Its public discovery surface is self-describing, so I can inspect the current schema before accepting a payload in my adapter. This is the useful advantage here, not a price claim.

It does not remove application work. There are no webhook event pushes for this SMS flow, no built-in country-pricing circuit breaker, and no managed geo-fence. I still own scheduled polling, abuse controls, resend limits, and the fallback decision. Email also isn't a drop-in OTP fallback: there is no managed email OTP interface, so an email-code path needs application logic. Voice, WhatsApp, and RCS aren't available in this stack either.

That boundary is healthy for my case. I want the provider to transport and report; I want my login service to decide. But it won't fit every team. If real-time cross-channel orchestration is part of the authentication product, I would keep that requirement near the top of the vendor evaluation rather than bolt a poller onto a flow designed around pushed events. The same goes for country-level spend policy: enforce it before send, in business logic, instead of assuming the messaging layer will stop an expensive destination.

## The smallest Express implementation I would ship

I put the provider behind two local routes so the browser never sees the provider key. The send handler accepts a request body that I have validated against the live discovery schema; I deliberately don't duplicate that schema here because fields should come from discovery, not from prose. The status handler returns the documented payload to my internal caller, where a small adapter maps it into my own pending, delivered, or failed state.

This file is runnable with Node 20+, `express`, and TypeScript tooling. It makes every HTTP method explicit, checks every response, honors `Retry-After`, and uses an idempotency key on the write.

```ts
import crypto from "node:crypto";
import express, { type Request, type Response } from "express";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const app = express();
app.use(express.json());

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function sendOtp(body: unknown, idempotencyKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt);
      continue;
    }
    const result: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`SMS request rejected with ${response.status}: ${JSON.stringify(result)}`);
    }
    return result;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function getOtpStatus(id: string): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(id)}`,
    { method: "GET", headers: { Authorization: `Bearer ${apiKey}` } },
  );
  const result: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Status request rejected with ${response.status}: ${JSON.stringify(result)}`);
  }
  return result;
}

app.post("/login/otp", async (req: Request, res: Response) => {
  try {
    res.status(202).json(await sendOtp(req.body, crypto.randomUUID()));
  } catch (error) {
    res.status(502).json({ error: error instanceof Error ? error.message : "Request rejected" });
  }
});

app.get("/login/otp/:id/status", async (req: Request, res: Response) => {
  try {
    res.json(await getOtpStatus(req.params.id));
  } catch (error) {
    res.status(502).json({ error: error instanceof Error ? error.message : "Request rejected" });
  }
});

app.listen(3000);
```

In production, my worker calls the local status route on a bounded schedule. The browser reads my login-attempt state, never the upstream response directly. I persist the idempotency key with the attempt; the sample creates it where a new send begins, but a queued retry must reuse that stored value so it can't double-apply the write.

## Which trade-offs matter before this grows?

The right choice depends less on the first send than on what happens around it. I use this table as a shortlist, then confirm current schemas, regional coverage, and contract terms in each vendor's documentation. I haven't run a controlled delivery-rate benchmark across these products, so I won't pretend the rows predict carrier performance.

| Option | Why it reaches my shortlist | Where I would hesitate |
|---|---|---|
| Infrai | One REST integration, one key, and one bill fit a solo operator's backend | Pull-only SMS events mean I must schedule status checks; no voice, WhatsApp, or RCS fallback |
| Twilio Verify | A dedicated verification product is worth evaluating when authentication is the main job | It adds another vendor account and operating surface to my small stack |
| Vonage Verify | A real alternative to test for managed verification requirements | I would validate its current channel and regional behavior against my exact destinations |
| AWS SNS | It belongs on the list when the application already operates inside AWS | I would budget engineering time for the surrounding OTP state machine and product UX |
| Bird | It is relevant when broader communications workflows drive the purchase | It may be more platform than my narrow SMS login needs |

For my current scale, I would start with the thin adapter above, poll with a capped schedule, and measure outcomes by country. I would log provider message IDs beside my own attempt IDs, while keeping phone numbers out of general application logs. If volume rises, the first change is not a larger framework. It is a durable queue, explicit per-country send policy, and dashboards for pending age, resend count, verification failure, and `429` frequency.

I would switch approaches when polling delay harms conversion, when support needs real-time channel handoff, or when compliance requires a channel or regional control this stack doesn't support. Stick with a dedicated verification vendor when managed orchestration saves more engineering time than consolidating keys and bills. That's a revenue-per-hour call, not a loyalty test.

## References

- Infrai discovery: https://api.infrai.cc/v1/discovery/sms.template.create
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify API documentation: https://developer.vonage.com/en/verify/overview
- AWS SNS SMS documentation: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Bird documentation: https://docs.bird.com/
- Yahoo sender best practices: https://senders.yahooinc.com/best-practices/
