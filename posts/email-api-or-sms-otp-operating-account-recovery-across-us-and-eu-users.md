# Email API or SMS OTP? Operating Account Recovery Across US and EU Users

Bottom line: I use a single-use email link as the default account-recovery path for a small SaaS, and I add SMS OTP only when a verified phone number already belongs in the product. Email keeps the recovery state machine and the support burden smaller; SMS can shorten the path for phone-first users, but it adds message segmentation, phone-number lifecycle, and another delivery channel to operate.

| Choice | Best default when | Main operational burden | Cost behavior |
|---|---|---|---|
| Email reset link | Email is the account identifier | Sender authentication, inbox delivery, link expiry | One send per attempt is the useful planning unit |
| SMS one-time code | A verified phone is already part of the account | Number changes, code entry, segment count | Each SMS segment is a billable-risk unit |
| Both channels | The product can justify two tested recovery paths | Duplicate policy, monitoring, abuse controls, support | Two separate demand curves |

My recommendation sits under one constraint: recovery should follow identity data I already maintain. I don't collect phone numbers merely to create a backup channel. For a one-person company shipping weekly, every extra recovery branch competes with customer work, and revenue per engineering hour matters more than a polished channel menu.

## Which password reset channel is simpler for SaaS login recovery?

Email wins my simplicity test when signup already requires an address. The user asks for recovery, the application creates an opaque single-use token, stores only a digest with an expiry, and sends a link. On return, the server consumes the token and invalidates existing recovery tokens for that account. The browser is already where the new password belongs.

One loop closes.

An SMS OTP looks shorter on a mockup, but the backend still needs issuance, expiry, attempt limits, one-time consumption, and a final credential-change step. It also needs a reliable association between an account and a verified number. Numbers get reformatted, replaced, or shared; those are data-model and support questions I must answer before the first message goes out. I can't outsource the policy.

Email has real work too. DomainKeys Identified Mail, specified by RFC 6376, lets a signer take responsibility for a message by attaching a domain-linked cryptographic signature. In practice, I treat sender authentication and alignment as deployment work, not a line item to revisit after users complain. I also keep the recovery response identical for known and unknown addresses, so the request screen doesn't disclose membership. The useful architectural split is a channel-neutral recovery service plus a narrow delivery adapter. The service owns token state and abuse controls; the adapter owns message construction and transport response metadata. For me, “simpler” means fewer states that can wake me up or create a support ticket, not fewer fields on one screen. By that measure, one channel tied to the existing login identifier is the clean default.

## The criterion I care about first: tail latency under actual traffic

Recovery latency isn't the median send time. It's the time between a user's request and the moment they can continue, at the slow edge of the distribution. That path crosses my application, a queue or worker, a delivery network, and either an inbox or a handset. I measure request-to-accepted and request-to-consumed separately. Otherwise a fast API response can hide a user waiting for the actual message.

I learned this once. A newly scaled-to-zero recovery worker looked fine in staging, then real traffic pushed its p99 request-to-queue time from 180 ms to 4.8 seconds during a Monday burst. The median barely moved. I'm not sure why the warm-up profile differed so sharply from my load test, but the fix in my operating model was clear: exercise the exact deployed path, preserve percentile telemetry, and alert on the user-visible budget rather than average latency.

That changed my checklist. I now deploy recovery code behind a synthetic test that requests a token for a dedicated account, observes delivery through the test adapter, consumes it, and confirms reuse fails. Production monitoring records a correlation ID, channel, enqueue duration, provider acceptance class, and consumption delay without logging the token or password. I retain enough aggregate data to compare regions, but I don't put email addresses or phone numbers in metric labels.

Keep it boring.

One policy.

US and EU users don't justify separate security semantics. They may justify separate telemetry cuts, sender configuration, and data-retention decisions. Your mileage may vary because traffic shape and channel coverage depend on the customer base. I still want one token policy and one consumption path — the channel should not decide whether a credential can be changed.

## The second criterion: cost is a workload shape, not a sticker price

I forecast recovery cost from attempts, retries, abuse, and message units. I don't use a single advertised rate as the decision. That rate can change, and it says little about the engineering time spent on a second channel.

SMS makes the unit especially visible. A GSM-7 message can hold up to 160 characters, while UCS-2 reduces the single-message limit to 70. Concatenated messages use smaller per-segment limits because headers consume space. A translated label, curly punctuation, or long warning can therefore change the number of segments. I keep an OTP message deliberately plain, count its encoding and segments in a test, and treat localization as a release that can alter spend. Short copy is an operational control here, not a branding exercise.

For email, I model sends per recovery request and watch retries, bounces, and repeated requests. For either channel, abusive requests can dominate legitimate demand, so I put coarse rate controls at the account, destination-digest, and network edge. Those controls should return the same public response regardless of whether an account exists. Internally, I need a reason code that distinguishes suppressed, queued, accepted, and consumed events without exposing sensitive destinations.

The bigger cost is mine. If a second channel takes half a day each month to test, reconcile, and support, it has to beat the feature I could ship in that half day. I run a one-person SaaS; “cheap” infrastructure that expands my pager and policy surface is expensive. I outsource transport because it is undifferentiated, but I keep recovery policy in my code so I can switch delivery adapters without rewriting credential logic.

## A small TypeScript boundary keeps the implementation honest

The implementation I want is channel-neutral up to one dispatch call. This sketch omits framework wiring, persistence details, and the actual transport client, but it shows the boundary I test. The application creates a short-lived secret, stores only its digest, and passes a rendered destination to an adapter. Consumption is atomic: exactly one request can move a token from active to used.

```ts
import { createHash, randomBytes } from "node:crypto";

type Delivery = {
  sendPasswordReset(input: {
    destination: string;
    resetUrl: URL;
    correlationId: string;
  }): Promise<void>;
};

type RecoveryStore = {
  replaceActive(input: {
    accountId: string;
    tokenDigest: string;
    expiresAt: Date;
  }): Promise<void>;
};

export async function requestPasswordReset(input: {
  accountId: string;
  email: string;
  publicOrigin: URL;
  delivery: Delivery;
  store: RecoveryStore;
}): Promise<void> {
  const token = randomBytes(32).toString("base64url");
  const tokenDigest = createHash("sha256").update(token).digest("hex");
  const expiresAt = new Date(Date.now() + 15 * 60 * 1000);
  const correlationId = randomBytes(12).toString("hex");

  await input.store.replaceActive({
    accountId: input.accountId,
    tokenDigest,
    expiresAt,
  });

  const resetUrl = new URL("/auth/reset", input.publicOrigin);
  resetUrl.searchParams.set("token", token);

  await input.delivery.sendPasswordReset({
    destination: input.email,
    resetUrl,
    correlationId,
  });
}
```

The public request handler still returns one generic acknowledgement and does the account lookup away from the response body. My tests freeze time, assert the 15-minute boundary, race two consumption calls, and verify that only one succeeds. They also assert that logs never contain the raw token. Those checks earn their keep every release.

The catch is that email isn't suitable as the only route when the product is genuinely phone-first, users may lack dependable inbox access, and a verified number is already maintained for core product behavior. In that case, use SMS as the primary recovery adapter and keep the same server-side token rules. A two-channel design is justified when support evidence shows that one existing identifier routinely becomes unavailable and the business can operate both paths. Don't add both by instinct: each route needs equivalent abuse tests, telemetry, deployment checks, and a documented way to update its destination safely.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- SMS character limits and GSM-7/UCS-2 segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
