# How to Audit a Node.js SaaS Email API (Custom DKIM and SPF)

Short answer: choose a transactional email API only after a custom-domain test proves DKIM alignment, region handling, template portability, and duplicate-safe delivery for the exact Node.js workflow that sends an order receipt after payment settles.

| Choice | Delivery control | Setup burden | Best fit |
| --- | --- | --- | --- |
| Managed HTTPS API | Domain authentication, provider event data, templates | Low to medium | A small SaaS shipping weekly |
| Cloud email primitive | Fine-grained infrastructure and regional choices | Medium to high | Teams already operating that cloud |
| SMTP relay | Broad client compatibility | Medium | Existing software that can't call HTTPS |

For a one-person logistics SaaS, the default is the managed HTTPS API. The runner-up is a cloud email primitive when regional architecture or infrastructure ownership matters more than setup time. SMTP belongs in the matrix, but it isn't the default for a new Node.js service.

## How can a Node.js SaaS test a transactional email API?

Test the path, not the feature grid. A provider can advertise a custom domain, DKIM, SPF, a template API, and US/EU service while still being a poor match for an order receipt. The useful test begins with a `payment.settled` event and ends with evidence that one message, bearing the correct order data, reached a controlled mailbox. It also records what happens when the worker receives that event twice.

The first gate is identity. SPF authorizes sending hosts, DKIM signs the message, and DMARC evaluates alignment and tells receiving systems how the domain owner wants failures handled. These are related controls, not three interchangeable checkboxes. Use a dedicated sending subdomain, publish the records the candidate supplies, then inspect the received message headers. Don't promote the integration because a dashboard shows a green badge.

Headers decide.

The second gate is data location and event access. “Available in Europe” can describe a sales presence, an API edge, or actual message processing; those aren't the same promise. Ask where message bodies, recipient addresses, templates, logs, and backups are processed and retained. Put the answer in the architecture decision record. I'm not sure a generic regional label resolves any particular company's legal obligations; counsel and the provider's current data-processing terms resolve that.

## Budget the duplicate-delivery work before comparing features

Email APIs accept work over a network. Payment systems also retry events. If the request handler sends immediately, a timeout can leave the application unable to tell whether the receipt was accepted, and a replay can create a duplicate. The safer boundary is a transactional outbox: commit the settled payment and an `order_receipt.requested` record together, then let a worker deliver it. This keeps the revenue path short and moves undifferentiated retry work out of the checkout request.

Exactly-once delivery isn't a realistic end-to-end assumption here. Build for at-least-once processing and make the effect duplicate-safe. Walk through one concrete failure before committing to a candidate: order `LGX-1842` settles, the outbox worker posts the receipt, and its 10-second client deadline expires before the acceptance response is stored. The next attempt cannot infer delivery from that timeout. It first checks a send ledger keyed by `order-receipt:LGX-1842`; if no accepted message ID exists, it retries with the same stable key, records the returned ID in the same database transaction that marks the job delivered, and leaves later attempts with nothing to send. If the selected API documents an idempotency mechanism, map the stable key to it as a second guard. If it doesn't, the local ledger remains the guard you control. This drill also exposes a weak event model: a payment event without an immutable order ID, recipient snapshot, currency, and settled amount cannot reproduce the receipt safely after customer data changes.

Keep retries narrow. Retry network failures and documented transient responses with exponential backoff plus jitter. Don't retry a malformed recipient or an authentication rejection forever. A dead-letter state needs the order ID, attempt count, last classified error, and next operator action — never the full email body when identifiers will do.

This is the boring part. Good.

## Rollout the TypeScript adapter and send ledger

Templates should produce a versioned subject and body before the provider adapter runs. That separation keeps order fields, escaping rules, and snapshot tests in the application repository; swapping an API then changes one adapter instead of every call site. The example below uses Node.js built-in `fetch`, validates required configuration, passes a stable idempotency key when the chosen API documents such a header, and treats acceptance as a state to record rather than proof of inbox placement.

```ts
type OrderReceipt = {
  orderId: string;
  recipient: string;
  amount: string;
  trackingUrl: string;
};

type AcceptedMessage = { messageId: string };

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function renderReceipt(receipt: OrderReceipt) {
  return {
    from: required("RECEIPT_FROM"),
    to: receipt.recipient,
    subject: `Receipt for order ${receipt.orderId}`,
    text: [
      `Payment received: ${receipt.amount}`,
      `Track the shipment: ${receipt.trackingUrl}`,
    ].join("\n"),
    metadata: { order_id: receipt.orderId, template_version: "receipt-v3" },
  };
}

export async function sendReceipt(receipt: OrderReceipt): Promise<AcceptedMessage> {
  const response = await fetch(required("EMAIL_SEND_ENDPOINT"), {
    method: "POST",
    headers: {
      authorization: `Bearer ${required("EMAIL_API_TOKEN")}`,
      "content-type": "application/json",
      "idempotency-key": `order-receipt:${receipt.orderId}`,
    },
    body: JSON.stringify(renderReceipt(receipt)),
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    throw new Error(`Email request rejected with status ${response.status}`);
  }

  const result = (await response.json()) as { messageId?: string };
  if (!result.messageId) throw new Error("Email response omitted messageId");
  return { messageId: result.messageId };
}
```

`EMAIL_SEND_ENDPOINT`, the authorization shape, request schema, response schema, and idempotency header must match the selected provider's current documentation. That's deliberate: a fake “universal” payload hides the most expensive portability work. Put those details behind this function and contract-test the adapter in a staging account.

Run three deployment checks. First, render fixtures containing a long customer name, a missing optional tracking value, and HTML-like characters; snapshot the text and HTML variants if both are sent. Second, send to controlled mailboxes at more than one mailbox provider and retain authentication results from headers. Third, replay the same outbox job and confirm the send ledger still maps the order to one accepted message ID. Track acceptance, delivery, bounce, complaint, and time-to-delivery as distinct states. Acceptance alone isn't delivery.

## Regional data evidence beats regional labels

Real candidates expose different operating models. Amazon Simple Email Service is a cloud email primitive with region-specific endpoints and identities, which suits an architecture already partitioned by AWS Region but adds more assembly work. Postmark separates message streams and documents server-side templates, a more opinionated model that can reduce application plumbing. Twilio SendGrid documents domain authentication and dynamic templates, while Mailgun documents separate US and EU API base URLs. Those facts help form a shortlist; none proves reliable delivery for your domain and recipient mix.

The catch is that a managed HTTPS API is not suitable when a legacy ERP can emit only SMTP, when policy requires self-hosted mail transfer, or when a company's approved cloud boundary dictates a specific regional primitive. Stick with SMTP for the first case, operate the mail stack for the second only if the team accepts abuse handling and deliverability operations, and use the approved cloud service for the third. A simple setup loses its value when it violates the actual deployment constraint.

Vendor templates are also a trade. They let non-developers edit copy and may speed an urgent correction, but they put production behavior outside the application commit that triggered it. Repository-owned rendering is easier to review, test, and migrate. For a solo operator shipping weekly, that predictability usually earns more revenue per hour than a visual editor. A content-heavy team with frequent non-engineering edits may reasonably choose the opposite.

Before signing off, require a written exit test: export the active templates, suppressions, and event history in a usable form; identify how sending-domain DNS changes during migration; and estimate how long both providers must run in parallel. Your mileage may vary because retention windows and export formats change. Verify them in the current contract and docs, not an old comparison post.

## Cost the operating model before selection

Choose the smallest operating model that passes the domain-authentication, regional-processing, duplicate-delivery, and event-evidence tests. For the logistics receipt flow, that usually means an HTTPS API behind a narrow adapter and a durable outbox. It does not mean choosing the service with the longest feature page.

Ship the test before the integration. If a candidate can't produce inspectable headers, a stable acceptance identifier, a documented failure taxonomy, and a credible migration path, stop there. The weekly roadmap has better uses for that hour.

## References

- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/ses/latest/dg/regions.html
- https://postmarkapp.com/developer/user-guide/templates/templates-overview
- https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication
- https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages/post-v3--domain-name--messages
