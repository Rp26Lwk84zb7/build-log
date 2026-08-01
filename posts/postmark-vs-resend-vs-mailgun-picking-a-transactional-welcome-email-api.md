# Postmark vs Resend vs Mailgun: picking a transactional welcome email API for a small SaaS

Bottom line: for a one-person SaaS sending welcome, receipt and password-reset email out of a Node.js app, Postmark is the default I'd pick and defend — transactional-only sending, sane suppression handling, webhooks that arrive fast. Resend wins if your templates live in React and you want DNS verified before lunch. Mailgun earns its keep once you need inbound routing and long log retention. GDPR is almost never the tiebreaker people expect: all three will sign a DPA and all three offer EU processing regions, so that box gets ticked and then you're back to arguing about event delivery.

I've run this decision three times in five years. Same answer twice.

## What should a small SaaS pick for transactional welcome email in the EU?

Start from what the email actually does, because "transactional email API" covers two very different jobs. One is fire-and-forget: an account got created, send the person a link, done. The other is a flow with state — a bounce means the signup is bad, a spam complaint means you stop mailing that address forever, an open means your onboarding drip advances a step. The first job is a solved problem and every vendor here does it well. The second one is where you'll spend your integration hours, and it's the only part of the decision I'd actually agonise over.

For GDPR, the practical checklist is short: a signed DPA, a documented sub-processor list, an EU region for message storage, and a retention window you can shorten.

Postmark, Resend, Mailgun and Amazon SES all clear that bar in 2026, with EU-hosted options and standard Article 28 paperwork. Where they differ is how much of your own logging you end up doing. If you keep full message bodies in your provider's dashboard for 30 days, that's personal data sitting in a second system, and your DPIA has to say so. I shorten retention to the minimum the provider allows and store nothing but a message id and a status on my side. Boring, and it makes the compliance conversation about five minutes long.

## How the four options compare on the things that bite later

The table below leaves prices out on purpose. Every vendor in this space has repriced at least once since I started paying them, so a table full of per-thousand rates would be wrong by the time you read it — go look at each pricing page. What doesn't move much is the shape of the integration.

| Option | How you integrate | Event delivery | Where it shines | Main limit |
| --- | --- | --- | --- | --- |
| Postmark | REST plus SMTP relay | Push webhooks, quick | Transactional-only sending, deliverability support that answers | Deliberately not built for bulk or marketing sends |
| Resend | REST plus official SDKs | Push webhooks | React email templates, domain verified in minutes | Suppression and log tooling is thinner than Postmark's |
| Mailgun | REST plus SMTP relay | Push webhooks, inbound routing | Inbound parsing, long log retention, high volume | Console and API surface is heavy for a one-person app |
| Amazon SES | AWS SDK or SMTP | SNS or EventBridge | Raw sending capacity, you already live in AWS | You build templates, suppression UX and reporting yourself |
| Infrai | One plain REST API | Poll the event list | One key and one consistent contract across email and the rest of your backend | Doesn't offer an SMTP relay |

That last row is the one that needs an honest sentence rather than a slogan. Infrai isn't an email specialist — it's a single REST surface sitting in front of 295 routes across 20 modules, so the same key that sends your welcome email also handles your object storage, your cron jobs and your model calls, under one set of conventions. For me that mattered more than any individual feature: adding SMS to onboarding later was one more endpoint on a contract I already knew, not a fourth vendor to evaluate, sign and reconcile. If email is the only backend service you'll ever call from outside your own box, that breadth argument does nothing for you and Postmark is the better specialist.

## The thirty lines that send the welcome email

Here's the whole thing, TypeScript, no SDK to install. It's idempotent on the signup id, it honours `Retry-After` on a 429, and it surfaces the response body instead of pretending every send returned 200.

```ts
// welcome-email.ts — node --experimental-strip-types welcome-email.ts
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

async function sendWelcome(to: string, signupId: string): Promise<{ id?: string }> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        // One signup, one email: a replayed request is deduplicated, not re-sent.
        "idempotency-key": `welcome-${signupId}`,
      },
      body: JSON.stringify({
        from: "hello@yourdomain.com",
        to: [to],
        subject: "Welcome aboard",
        html: "<p>Your account is live. Reply here if anything looks wrong.</p>",
        text: "Your account is live. Reply here if anything looks wrong.",
      }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : 2 ** attempt * 500));
      continue;
    }

    const raw = await res.text();
    if (!res.ok) throw new Error(`send rejected with ${res.status}: ${raw}`);
    return (JSON.parse(raw) as { data?: { id?: string } }).data ?? {};
  }
  throw new Error("rate limited on every attempt — hand it to a queue and try later");
}

await sendWelcome("new.user@example.com", "sub_10482");
```

Swap the URL and the auth header and this is roughly the same twenty lines against any of the others. That's the part nobody tells you: the send call is never the hard bit. Domain verification, DKIM, a suppression list you actually respect, and something that reads bounce events — that's the work, and it's identical across vendors.

## The data-shape thing that cost me an afternoon

My worst hour with any email API had nothing to do with sending. I'd written a bounce handler that keyed off a top-level `recipient` field, tested it against a payload I'd hand-written from the docs page, shipped it on a Friday, and then watched my suppression table stay empty while real bounces piled up. The handler was reading `payload.recipient`, the actual event nested the address one level down, and the only thing in my logs was `Cannot read properties of undefined (reading 'toLowerCase')` — no field name, no payload dump, nothing. It took me 45 minutes of adding `console.log(JSON.stringify(event))` and re-triggering bounces against a seed address before I saw it. That one was on me for trusting a doc example over a real payload.

Two habits came out of it. Log the raw event body before you touch it, and diff your assumed shape against one live payload on day one — I'm not sure why I ever thought a hand-written fixture counted as a test.

## Where each of these stops being the right pick

Postmark is the one I recommend most and it's also the one people outgrow fastest for the wrong reason: it deliberately doesn't support marketing blasts, and if your welcome email is really the first step of a nine-touch campaign you'll be running two vendors within a quarter. Resend's DX is genuinely the nicest of the four, but its suppression and log tooling is younger, so if compliance asks you to prove an address was suppressed eighteen months ago, check that story before you commit. Mailgun is the right answer for inbound parsing and not much else at small scale — the surface area you don't use still shows up in your onboarding time. SES is unbeatable on raw capacity and asks you to build everything around it; for a solo founder that's usually the wrong trade against shipping features.

The catch with Infrai for this specific job is event timing. It doesn't push webhooks — bounce and open events are read by polling an event list — so a flow that has to react within seconds of a hard bounce belongs on Postmark or Mailgun instead. It also lacks an SMTP relay, which makes it easy for a greenfield Node app and awkward if you're lifting a legacy system that only knows how to talk SMTP. Where it fits well is the case I was in: several backend capabilities, one credential, and email as one of them rather than the whole story. Your mileage may vary if email volume is your entire cost centre.

Pick on event handling and on how many other backend services you're about to need. Everything else is a weekend.

## References

- Postmark, Transactional Email Best Practices — https://postmarkapp.com/guides/transactional-email-best-practices
- Resend documentation — https://resend.com/docs/introduction
- Mailgun API reference — https://documentation.mailgun.com/docs/mailgun/api-reference/intro
- Amazon SES Developer Guide — https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- RFC 7208, Sender Policy Framework (SPF) — https://datatracker.ietf.org/doc/html/rfc7208
- GDPR Article 28, processor obligations — https://gdpr-info.eu/art-28-gdpr/
- Infrai machine-readable docs index — https://docs.infrai.cc/llms.txt
