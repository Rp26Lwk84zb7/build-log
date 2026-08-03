# Picking an SMS alerts provider for appointment reminders and account activity

## TL;DR

Twilio is the provider I still reach for when one small app has to push appointment reminders, shipping alerts and account-activity notices to both US and EU numbers, because it's the one that hands you carrier registration, opt-out handling and delivery receipts without a sales call. I'd move off it for two specific reasons: Plivo or Vonage when per-message cost dominates a high-volume flow, and AWS End User Messaging SMS when the rest of the stack already lives in one AWS account. The simple REST API is the easy part — country-by-country registration and suppressions are what sink a launch, and templates belong in your repo, not in a vendor console.

I run a one-person SaaS. Any hour I spend on messaging plumbing is an hour not spent on the features people actually pay for, so I grade providers on two things: how fast I get a correct alert out the door, and how much of next month they quietly claim back.

Three flows, one pipe.

Appointment reminders are scheduled and forgiving; if one lands ninety seconds late, nobody files a ticket. Shipping alerts are event-driven and bursty, which means the send path has to survive a carrier webhook storm without double-texting anyone. Account-activity notices — new sign-in, password change, payout sent — are the ones customers screenshot and forward to support when they don't arrive, so they get the highest retry budget and the loudest alarm on my side. One send function, three failure budgets.

## What should I look for in an SMS alerts provider for appointment reminders and account activity?

Coverage, in the boring regulatory sense, before anything else. In the US, application-to-person traffic on a normal 10-digit number needs 10DLC registration: you register a brand and a campaign through The Campaign Registry via your provider, and unregistered traffic gets filtered by carriers rather than politely rejected. Toll-free numbers need their own verification. In much of Europe none of that applies, but alphanumeric sender IDs do, and several countries — France notably requires pre-registration of the sender ID — have their own paperwork. A provider that walks you through both from one dashboard is worth real money to a solo founder.

Second, delivery receipts. Any provider worth using will POST status transitions to a webhook you own: queued, sent, delivered, undelivered, failed, with a numeric error code attached. Store every one of those against the message row. That single table is what lets you answer "did the reminder go out?" in ten seconds instead of grepping vendor logs.

Third, suppressions. Someone replying STOP has to stop receiving messages permanently, and you need that state readable by your own code — a provider-internal list you can't query is a compliance risk you'll discover on the worst possible day.

Templates come fourth, and honestly they matter less than the vendor marketing suggests. Most SMS APIs let you POST a plain body string; the ones that offer server-side templates are mostly solving for WhatsApp approval flows, not for your reminder copy. I keep message bodies in a TypeScript file next to the code that sends them, so a copy change is a pull request with a diff and a deploy, not an undocumented edit in someone's console.

## The five options I'd actually shortlist

| Option | API shape | Best fit | The catch |
| --- | --- | --- | --- |
| Twilio | Form-encoded REST, one POST per message, Basic auth | Mixed US + EU alert traffic where you want registration and opt-out handled for you | The most surface area to learn: services, senders, campaigns and messages are all separate objects |
| Vonage | REST with key/secret, JSON or form bodies | Straightforward one-way alerts, good European reach | Thinner ecosystem of examples; less orchestration built in |
| Plivo | REST, JSON bodies, Basic auth, Twilio-shaped primitives | High volume where unit cost drives the decision | Fewer hand-holding features around registration; docs get thin in the corners |
| AWS End User Messaging SMS | AWS SDK or signed API, IAM auth | Teams already fully inside AWS wanting one account and one bill | Regional origination identities and a low default spend quota you must raise before real traffic |
| Courier | One API over your chosen providers, with templates and preferences | When per-user channel preferences and template versioning are the real problem | Another vendor in the path, and you still need a real SMS provider underneath |

Sinch and Bandwidth belong in the conversation at volumes I don't operate at — direct carrier relationships, better rates, sales-led onboarding. If you can't sign up and send a test message in an afternoon, they're not a good fit for a one-person shop, and I'd stick with a self-serve provider until someone else is paying for the migration.

The comparison that matters isn't feature-by-feature anyway. It's how many of these you'd need to run at once, and the answer for most alert workloads is one, plus an email fallback.

## Sending one alert, end to end

Here's the whole send path I use, trimmed of my logging. Suppression check, dedupe check, POST, persist:

```ts
// alerts/sms.ts — one send path for reminders, shipping and account activity.
const SID = process.env.TWILIO_ACCOUNT_SID!;
const TOKEN = process.env.TWILIO_AUTH_TOKEN!;
const SERVICE = process.env.TWILIO_MESSAGING_SERVICE_SID!;

type Alert =
  | { kind: "appointment_reminder"; when: string; clinic: string }
  | { kind: "shipping_update"; carrier: string; tracking: string }
  | { kind: "account_activity"; event: string; city: string };

// Message copy lives in the repo, so changing it is a PR someone can review.
function render(a: Alert): string {
  switch (a.kind) {
    case "appointment_reminder":
      return `Reminder: ${a.clinic}, ${a.when}. Reply STOP to opt out.`;
    case "shipping_update":
      return `Your order shipped via ${a.carrier}. Tracking: ${a.tracking}`;
    case "account_activity":
      return `${a.event} on your account from ${a.city}. Not you? Reset your password.`;
  }
}

export async function sendAlert(to: string, alert: Alert, dedupeKey: string) {
  if (await isSuppressed(to)) return { skipped: "suppressed" as const };
  if (await alreadySent(dedupeKey)) return { skipped: "duplicate" as const };

  const res = await fetch(
    `https://api.twilio.com/2010-04-01/Accounts/${SID}/Messages.json`,
    {
      method: "POST",
      headers: {
        // Basic auth: account SID is the username, the token is the password.
        authorization: `Basic ${Buffer.from(`${SID}:${TOKEN}`).toString("base64")}`,
        "content-type": "application/x-www-form-urlencoded",
      },
      body: new URLSearchParams({
        To: to,
        MessagingServiceSid: SERVICE,
        Body: render(alert),
        StatusCallback: "https://api.example.com/hooks/sms-status",
      }),
    },
  );

  const json = await res.json();
  if (!res.ok) throw new SmsError(json.code, json.message); // json.code is the numeric provider code
  await recordSent(dedupeKey, json.sid);
  return { sid: json.sid as string };
}
```

The dedupe key is mine, not the provider's: `shipment:${id}:shipped`. Retries are cheap and duplicate texts are expensive, and as far as I can tell none of the mainstream SMS APIs give you an idempotency header the way payment APIs do, so that check stays on my side of the wire.

Now the confusing failure, which cost me most of a Thursday. I lifted the auth helper from an older worker of mine that talked to a different vendor, so the header went out as `Authorization: Bearer ${token}` instead of Basic. What came back was a 401 with error code 20003 and a complaint about an invalid username — and I had no username in play at all, so I spent 40 minutes convinced I'd copied the account SID wrong between staging and production. My retry wrapper only logged `res.status`, so the actual body never reached my logs. I assumed a credentials problem. It was an auth scheme problem. Two different bugs that look identical from the outside, and the fix was one line.

Test credentials are the other thing to wire up before you ship. Twilio's test SID and token never deliver a real message, and there are magic numbers that force each failure path: posting to `+15005550001` returns error 21211 for an invalid destination, which is exactly what you want in a CI test asserting your error handling.

```bash
curl -X POST "https://api.twilio.com/2010-04-01/Accounts/$TEST_SID/Messages.json" \
  -u "$TEST_SID:$TEST_TOKEN" \
  --data-urlencode "To=+15005550001" \
  --data-urlencode "From=+15005550006" \
  --data-urlencode "Body=ping"
```

## Suppressions, STOP keywords, and the European wrinkle

US carriers require you to honor STOP, and to answer HELP. Twilio's messaging services can handle those keywords for you, and once a number opts out, further sends to it are rejected with error 21610 rather than silently dropped. That's the behavior you want: a loud, specific failure your queue can classify.

Keep your own suppression table regardless.

Opt-outs reach you through channels the provider never sees — a support email, a churned account, a phone number that got recycled to someone else. My table is three columns: number, reason, timestamp. The send path reads it before every message, which is the `isSuppressed` call above, and the provider-side list is a second net rather than the only one.

Europe is where the model changes shape. Alphanumeric sender IDs are widely supported and look far better in someone's inbox than a random long number, but they're one-way: the recipient cannot reply, so STOP-by-reply doesn't exist as a mechanism. If you rely on it for compliance, that's a design that quietly doesn't work outside North America. For EU traffic you either send from a real number that can receive replies, or you put an unsubscribe link and a named sender in the body — and you get consent right at signup, because GDPR doesn't care that your message was transactional in intent.

Account-activity alerts have one more constraint worth stating plainly: don't put anything in them that a stranger holding the phone shouldn't learn. OWASP's guidance on password-reset flows is the short version — no account enumeration, no secrets in the body, short-lived codes. "New sign-in from Lyon" is fine. "New sign-in to tony@example.com, code 448120" is a leak.

## Where SMS is the wrong tool

SMS earns its cost for time-sensitive, single-fact messages. For digests, receipts and anything a customer might want to search later, email is better and cheaper — I use Postmark for transactional mail, and Resend or Amazon SES are reasonable picks depending on how much of your own deliverability work you want to own. If you're sending both, read the receiving side's rules before you tune anything; Yahoo and Google publish sender requirements that are more prescriptive than most people expect. And if your users are all in an app you control, push notifications cost nothing per message. Your mileage may vary, but I've never regretted moving a non-urgent alert off SMS.

## References

- Twilio Message resource (REST API): https://www.twilio.com/docs/messaging/api/message-resource
- Twilio Advanced Opt-Out for messaging services: https://www.twilio.com/docs/messaging/services/advanced-opt-out
- Twilio test credentials and magic numbers: https://www.twilio.com/docs/iam/test-credentials
- Vonage SMS API overview: https://developer.vonage.com/en/messaging/sms/overview
- Plivo Message API: https://www.plivo.com/docs/sms/api/message
- AWS End User Messaging SMS user guide: https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
- The Campaign Registry (US 10DLC brand and campaign registration): https://www.campaignregistry.com/
- CTIA Messaging Principles and Best Practices: https://www.ctia.org/positions/messaging-principles-and-best-practices
- Courier documentation: https://www.courier.com/docs/
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Yahoo sender best practices and requirements: https://senders.yahooinc.com/best-practices/
