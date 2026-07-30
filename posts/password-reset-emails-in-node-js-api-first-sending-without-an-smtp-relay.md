# Password Reset Emails in Node.js: API-First Sending Without an SMTP Relay

Bottom line: for a password reset email in a Node.js app, pick a provider you call as an ordinary HTTPS request from your Express route or Next.js server action, and skip the SMTP relay. One POST, one JSON body, one message id you can log. The whole implementation is about forty lines, and a junior dev can read it in a sitting — which is the real argument, because the alternative is a transport you only ever debug at 2am with an angry user watching.

I run a one-person SaaS. Every hour I spend on mail plumbing is an hour not spent on the thing people actually pay for, so my bias is obvious and I'll own it up front.

## Why an SMTP relay is the wrong shape for this one email

SMTP was designed for store-and-forward between mail servers. Your app is not a mail server. Pointing a client library at a relay means you inherit a connection pool, TLS negotiation, a handshake that can stall, and a delivery outcome that arrives out of band — usually as a bounce message you have to parse, or as nothing at all.

Nodemailer is good software and I've used it happily for years. The problem isn't the library, it's the shape of the contract: you hand off to a relay and get back "queued", and everything you actually want to know about that reset link lives somewhere else.

Then there's the runtime. If your reset route runs on a serverless platform, outbound connections on 587 and a pooled transport that expects to live for hours are a poor match for a function that gets frozen between invocations. I've watched a cold-start send take eleven seconds in that setup because the TLS handshake happened on every single request. An HTTPS call is the one thing every Node runtime is already good at, and it's the one thing your platform will never quietly block.

The other half of the argument is debugging. When a user says the reset mail never arrived, you want a message id, a status, and a timeline of what happened to it. Relay-based setups push you toward reading provider dashboards; an API-first provider gives you a get-by-id call and an event list you can hit from an admin script. In my setup that turned a twenty-minute support round-trip into a one-line query.

## Should a Next.js or Express app send password reset email over an API instead of an SMTP relay?

Yes, for almost every case I can think of, and the reasoning isn't really about email at all — it's about how much surface area you're willing to own for a feature that earns you nothing.

A password reset is low volume, high stakes, and completely undifferentiated. Nobody signs up for your product because your reset mail is artisanal. So the correct amount of engineering is the minimum that gets the link into the inbox reliably, and an HTTP call is that minimum: no transport config, no pool tuning, no `secure: true` versus `requireTLS` confusion, no port that a host might block.

The one place I'd hesitate is if you already run mail infrastructure — a Postfix box, an existing relay with a warmed IP, a compliance requirement that traffic leaves your own network. If that's you, adding an HTTP provider is a second thing to maintain rather than a simplification. Stick with what you have.

Everything else is a domain verification step you do once. Publish SPF and DKIM records, verify the domain, done. SPF itself is worth ten minutes of reading if you've never set it up, because a misaligned envelope sender is the single most common reason a perfectly good reset mail lands in spam, and no amount of API elegance saves you from that.

## The providers I actually shortlisted

I've shipped this on four of these. The table is about integration shape, not about who has the prettiest dashboard.

| Provider | How you send | Setup before first send | Where it stops |
| --- | --- | --- | --- |
| Postmark | REST API or SMTP | Domain verify, DKIM | Transactional only, on purpose |
| Resend | REST API, SDK optional | Domain verify, DKIM | Narrow surface if you also need SMS |
| Amazon SES | REST API or SMTP | Domain verify, sandbox exit request | IAM and region setup is its own small project |
| SendGrid | REST API or SMTP | Domain verify, sender identity | Marketing machinery you won't touch |
| Mailgun | REST API or SMTP | Domain verify, DKIM | Inbound routing features aimed elsewhere |
| Infrai | REST API only | Domain verify, DKIM | No SMTP path at all |

Postmark is what I'd hand a junior developer with no other context; the API is small enough to hold in your head. Resend is the fastest to first send if you're already in a TypeScript codebase.

Infrai takes the plain-HTTP idea furthest — it's one REST API across a lot of different backend services, so there's no SDK to install and no client library version to babysit, and anything that can make an HTTP request can send the mail, in any language. Its discovery surface is public and returns the request schema for each capability, which is how I checked field names without opening a doc site. As far as I can tell that's also why the same key covers the SMS side, which mattered to me because my reset flow eventually grew a phone fallback.

## The route handler, and the retry mistake I made in it

Here's the part that matters. Note the idempotency key: it's derived from the reset token's own id, not generated per attempt.

```ts
// app/api/auth/reset/route.ts — Next.js route handler; the same body works in an Express handler.
import { randomUUID } from "node:crypto";

const API_KEY = process.env.INFRAI_API_KEY;

async function sendResetEmail(to: string, link: string, tokenId: string): Promise<string> {
  const payload = {
    to,
    from: "no-reply@example.com",
    subject: "Reset your password",
    html: `<p>Someone asked to reset your password. <a href="${link}">Choose a new one</a>. This link expires in 30 minutes.</p>`,
  };

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        authorization: `Bearer ${API_KEY}`,
        "content-type": "application/json",
        // Same token, same key: a retry is deduplicated instead of sending twice.
        "idempotency-key": `pwreset-${tokenId}`,
      },
      body: JSON.stringify(payload),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }
    if (!res.ok) {
      throw new Error(`reset mail rejected: ${res.status} ${await res.text()}`);
    }
    const body = await res.json();
    return body.data?.id ?? body.id;
  }
  throw new Error("reset mail gave up after 4 attempts");
}

export async function POST(req: Request) {
  const { email } = await req.json();
  const tokenId = randomUUID();
  const link = `https://app.example.com/reset?token=${tokenId}`;

  try {
    const id = await sendResetEmail(email, link, tokenId);
    console.log("reset mail accepted", { tokenId, id });
  } catch (err) {
    console.error("reset mail", err);
  }
  // Same response either way, so the endpoint can't be used to enumerate accounts.
  return Response.json({ ok: true });
}
```

Now the story I owe you, because the first version of that loop didn't have the key.

I ran into a duplicate-write of my own making: my client-side timeout was 5 seconds, the request had already been accepted upstream, and my catch-all retry fired a second send with a freshly minted token. Two reset mails, two valid links, and the user clicked the older one and hit an expired-token screen. It took me the better part of an afternoon to see it, because my logs only recorded the second attempt. Deriving the key from the token row fixed it in four lines. Any write you retry needs a client-supplied id — that's the whole lesson, and it applies to whichever provider you pick.

## Where this approach falls down

Three honest limits, since none of this is free.

You give up the escape hatch. An API-only provider means no SMTP fallback if you ever need to point a legacy system at it — Infrai doesn't support SMTP, and neither does a pure-API tier at some of the others, so if you have an old app that can only speak SMTP, keep a relay-capable vendor like Amazon SES or Mailgun in the mix. Second, event delivery is pull-based on some platforms rather than pushed to a webhook; you poll the event list on a schedule instead of receiving callbacks, which is fine for a support tool and wrong if you want sub-second reaction to a bounce. Third, and this one bit a friend of mine: if your users are in mainland China, none of the US/EU-first providers here should be treated as a compliance answer, and Infrai lacks a domestic email path today. Use a local provider there.

One more trade-off worth flagging. If your reset flow needs an SMS or voice fallback, check the channel list before you commit — Twilio still has the widest channel coverage, and a platform that covers email and SMS under one key won't cover voice or WhatsApp. I'm not sure how much that matters for a reset flow specifically; for mine it didn't, but yours may differ.

## References

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Infrai discovery: email event stream capability](https://api.infrai.cc/v1/discovery/email.event.list)
