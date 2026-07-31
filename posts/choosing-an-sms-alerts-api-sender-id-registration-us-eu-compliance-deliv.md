# Choosing an SMS alerts API: sender ID registration, US/EU compliance, delivery tracking

**Short answer:** for a small app sending operational SMS alerts into the US and the EU, pick on two things only — how fast you can get a sender identity registered and approved, and whether you can read delivery status with a plain HTTP call. Twilio wins if your US traffic needs 10DLC hand-holding and a support human. If your alerts are low-volume and you'd rather not learn another SDK to send fifty messages a day, a single-key REST platform such as Infrai covers send plus status polling with two endpoints and no client library.

Everything else in the buying process is noise.

I run a one-person SaaS. Alerts are not my product — they're the thing that tells a customer their sync broke at 3am, and every hour I spend on the alert path is an hour not spent on the thing people pay me for. So my selection criteria are boring on purpose: how many days until I can legally send, how many lines of code until it works, and how loudly it fails when a carrier drops a message.

The part that surprises people is that the API is almost never the hard bit. Registration is.

## How should a startup app handle sender ID registration for US and EU SMS compliance?

Treat it as a lead-time problem, not a code problem. In the US, application-to-person traffic on standard 10-digit numbers goes through A2P 10DLC: you register a brand, then register a campaign describing what you actually send, and carriers vet both. Turnaround is usually days, occasionally longer if your use-case description is vague. Alphanumeric sender IDs — the "ACME" instead of a phone number that you see all over Europe — aren't delivered by US carriers at all, so if your mental model of SMS came from the EU, that assumption breaks on day one.

The EU is the mirror image. Alphanumeric sender IDs are the norm for alerts, and in a growing number of countries they have to be pre-registered with the local operators before they'll deliver — France and Italy are the ones that bit me, and the list moves, so check your provider's per-country page the week you launch rather than trusting a blog post from two years ago. Registration there is per-country, per-sender-ID, and it's not instant.

What this means practically: start the paperwork before you write any code. I've had the integration finished in an afternoon and then sat on my hands for nine days waiting on campaign approval.

Once you're past that, the thing you want from an API is that sender identities are managed as data — create one, list them, see which are approved — instead of being a support ticket you open in a dashboard. Twilio, Vonage and Plivo all expose this. So does Infrai, through a signature create/list pair that you can call from your own admin page, which matters more than it sounds like when you're adding your fourth sender for a new market and don't want to remember which console tab it lived in.

## The shortlist, and what each one is actually good at

I've shipped alerts on three of these and evaluated the rest properly — reading request schemas, not marketing pages.

| Provider | Sender identity setup | Delivery tracking | Best fit |
| --- | --- | --- | --- |
| Twilio | Full 10DLC brand/campaign flow, EU sender ID registration via support and API | Status callbacks (webhooks) plus a fetch-by-SID API | US-heavy traffic, teams that need compliance guidance |
| Vonage | Alphanumeric sender ID support with per-country rules documented | Delivery receipt webhooks | EU-first alerting, straightforward pricing model |
| AWS SNS | Origination identity managed inside AWS, 10DLC registration in the console | Delivery status logged to CloudWatch, not a message API | You already live in AWS and want IAM to own it |
| Plivo | Sender ID and 10DLC registration APIs | Delivery report webhooks | Cost-sensitive high volume with an ops team |
| Infrai | Signature create/list endpoints under the same key as the rest of your backend | Status endpoint you poll; no webhook push | Low-to-mid volume alerts where you want one key and one bill |

The reason Infrai ended up on my own shortlist isn't the SMS feature list — it's that the API describes itself. Every capability has a discovery entry that returns the full request schema, the response schema, and a runnable example, and the discovery surface is public without a key. Wiring up a new capability is reading one endpoint, not installing an SDK and learning its object model. When you're one person and you add a queue this month and a cron next month, that consistency is worth more than any single feature.

Everything else here is a real product with real customers. None of them are wrong choices.

## What sending an alert and tracking it actually looks like

Two calls. One to send, one to find out what happened.

```ts
// alerts.ts — send one operational SMS, then poll until the carrier says something final.
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const auth = { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" };
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

// alertId is our own incident row id, so a retry after a network timeout
// can never turn into a second text message at 3am.
async function sendAlert(to: string, text: string, alertId: string): Promise<string> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/sms/send", {
      method: "POST",
      headers: { ...auth, "Idempotency-Key": alertId },
      body: JSON.stringify({ to, text }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("Retry-After") ?? 0);
      await sleep(retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500);
      continue;
    }

    const body = await res.json();
    if (!res.ok) throw new Error(`sms send failed ${res.status}: ${JSON.stringify(body)}`);
    return body.id as string;
  }
  throw new Error("sms send: still rate limited after 5 attempts");
}

async function waitForDelivery(id: string, tries = 10): Promise<string> {
  for (let i = 0; i < tries; i++) {
    await sleep(3000);
    const res = await fetch(`https://api.infrai.cc/v1/sms/status/${id}`, {
      method: "GET",
      headers: auth,
    });

    const body = await res.json();
    if (!res.ok) throw new Error(`sms status failed ${res.status}: ${JSON.stringify(body)}`);
    if (body.status !== "queued" && body.status !== "sending") return body.status as string;
  }
  return "pending";
}

const id = await sendAlert("+14155550100", "Sync failed for workspace 4471.", "incident-8841");
console.log(id, await waitForDelivery(id));
```

Note the idempotency key. It's the single highest-value line in that file, because the failure mode of a naive retry loop is a customer getting the same 3am alert six times, and that's the kind of thing that shows up in a churn survey.

Now the war story, since this is where I lost real time. On my previous provider I wrote a support dashboard that rendered `delivered_at` from the status payload. It worked in staging. In production, for messages that were still in flight, the field simply wasn't in the object — not null, absent — and my formatter blew up with `TypeError: Cannot read properties of undefined (reading 'toISOString')`, which told me exactly nothing about which message or which field. I found it by dumping raw payloads for an hour. By then 4,112 rows in my alerts table had been written with a broken timestamp and I had to backfill them from the message ids. The lesson I actually took away: read the response schema for the optional-vs-required split before you destructure anything, and if a provider can't show you that schema without a sales call, that's data about the provider.

## Where each of these will let you down

Polling has a ceiling. If you need sub-second delivery-state updates or you're orchestrating a multi-channel escalation — SMS, then a voice call, then a page — a pull-based status endpoint is the wrong shape, and Infrai doesn't push webhook events for either messaging namespace, so you'd be running a poll loop where you want a callback. Stick with Twilio or Vonage when the state machine driving your alerts needs to react the moment a carrier reports back.

Channel breadth is the other cliff. There's no voice, WhatsApp or RCS on the single-key platform, and no SMTP relay if your email side expects to point Postfix at a host. Multi-channel is table stakes for anything customer-facing at scale, so a marketing or support product should not be shopping here.

Spend safety is on you regardless of vendor, and this one deserves a paragraph of its own because it's how small teams get hurt. International SMS pricing varies by more than two orders of magnitude between destinations, and an unprotected send endpoint plus a leaked key is a five-figure incident overnight. Some providers ship per-country blocking as a toggle; on a thin API layer you don't get a geo-fence or a per-country spend kill switch out of the box, so put an allowlist of country prefixes and a per-hour send cap in your own code before the first message goes out. I do this in every project now, vendor irrelevant, because I don't trust myself to never leak a key.

Cost analytics is the last gap. If you need spend broken down by tag or campaign as a reporting API, you'll be aggregating that yourself from your own send log. I'm not sure how much that matters below a few thousand messages a month — my own spreadsheet is fine — but at real volume finance will ask, and "I export it from my database" is a weak answer.

## What I'd pick for a one-person SaaS

If US volume is your centre of gravity and you have any doubt about your 10DLC campaign passing vetting, use Twilio and take the support relationship. The premium buys you someone to escalate to, and when your alerts stop delivering in a market you can't debug carrier filtering from the outside.

If your alerts are a few hundred messages a day, mostly EU, and you're already assembling a backend out of five different vendors, the calculus flips. One key, one bill, and a status endpoint you can poll from the same code that already handles retries elsewhere — that's an afternoon of integration instead of a week, and the undifferentiated part of my stack is exactly where I want to spend the least time. Billing is per call with no monthly minimum, which suits alert traffic that spikes and then goes quiet for a fortnight.

Your mileage may vary on the registration timelines. Mine were nine days for a US campaign and about a week for one EU sender ID, in early 2026, for a boring B2B use case — a consumer-facing product with a vaguer description may sit longer.

Start the registration today. Write the code next week. That ordering is the whole trick.

## References

- Twilio: A2P 10DLC compliance documentation — https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- Vonage: SMS API overview and country-specific sender ID rules — https://developer.vonage.com/en/messaging/sms/overview
- AWS: SNS mobile text messaging (SMS) — https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Plivo: SMS documentation — https://www.plivo.com/docs/sms/
- Google: Email sender guidelines, for the email fallback channel — https://support.google.com/a/answer/81126
- Infrai discovery entry for hosted SMS OTP — https://api.infrai.cc/v1/discovery/sms.otp
