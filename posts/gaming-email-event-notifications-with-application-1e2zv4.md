# Gaming Email Event Notifications With Application Owned Templates and Bounce Suppression

Keep gaming notification templates in the application, and check recipient suppression immediately before handing each email to a transport. Template ownership is the deciding constraint: a match notification should retain its approved wording even when delivery happens after another weekly release.

Short answer: implement email event notifications in Node.js as a small pipeline: capture a game event, render a named template revision, check suppression, and submit through an injected transport. Feed permanent recipient failures back into the same suppression store. The TypeScript example below demonstrates that boundary with a match result and an invalid recipient.

Keep ownership explicit.

For a one-person SaaS, the useful question is revenue per hour of engineering attention. Own the player-facing language and the rules about who receives it. Outsource the undifferentiated delivery work. This note proposes an application design; its example identifiers, event payloads, and failure classifications are illustrative contracts, not a commercial API.

## Why should template ownership change the notification design?

Suppose a game emits a match-completed event for player `player-17`. A release changes the result email from revision `match-result-v1` to a future revision. If a delayed job renders whatever template happens to be current, replaying the original event can produce different copy. That makes a support question harder: which message was actually authorized by that event?

Pin the revision. Keep old revisions available for the lifetime of those jobs. The application then owns both the meaning of a match event and its rendering. A transport only needs an envelope containing the destination, subject, and body. This trades a small amount of repository maintenance for a reviewable relationship between game behavior and player communication.

There is a boundary worth keeping narrow. Template ownership does not require building a mail server, managing every delivery retry, or designing a visual editor. For a weekly shipping cadence, a plain-text template reviewed alongside its triggering event is a reasonable starting point. If nontechnical editors later need independent publishing, an external template workflow may fit better, provided jobs can still identify an immutable revision and preserve the rendered output used for submission.

The hard part is the second lifecycle: the recipient can become suppressed after the job was created. A pinned template must not imply pinned permission to send. Store the destination identity separately from the template revision and check its current eligibility at dispatch.

## How do you implement email event notifications with a suppression check?

The example uses a stable recipient ID for suppression and a notification ID for duplicate handling. It deliberately accepts one internal failure classification, `permanent-recipient-failure`. An ingress adapter must produce that classification only after authenticating feedback and matching it to a known send. Raw webhook fields should never directly authorize a suppression change.

Memory is temporary.

Run the demonstration sequentially in a TypeScript-capable Node.js setup; it illustrates application decisions, not durable queue semantics. The transport is injected so the application doesn't inherit a provider's template identifiers or event vocabulary.

```ts
type MatchNotification = {
  id: string;
  recipientId: string;
  to: string;
  matchId: string;
  template: "match-result-v1";
};

type Envelope = {
  to: string;
  subject: string;
  text: string;
};

type Transport = {
  submit(message: Envelope): Promise<{ receiptId: string }>;
};

type VerifiedFeedback = {
  receiptId: string;
  kind: "permanent-recipient-failure";
};

function createNotifications(transport: Transport) {
  const suppressed = new Set<string>();
  const submitted = new Set<string>();
  const recipientsByReceipt = new Map<string, string>();

  function render(event: MatchNotification): Envelope {
    if (event.template !== "match-result-v1") {
      throw new Error("Unsupported template revision");
    }
    return {
      to: event.to,
      subject: "Your match result is ready",
      text: `Match ${event.matchId} has finished. Open the game to view it.`,
    };
  }

  async function notify(event: MatchNotification) {
    if (submitted.has(event.id)) return "already-submitted";
    const message = render(event);
    if (suppressed.has(event.recipientId)) return "suppressed";

    const receipt = await transport.submit(message);
    recipientsByReceipt.set(receipt.receiptId, event.recipientId);
    submitted.add(event.id);
    return "submitted";
  }

  function applyVerifiedFeedback(event: VerifiedFeedback) {
    const recipientId = recipientsByReceipt.get(event.receiptId);
    if (recipientId === undefined) return "unmatched";
    suppressed.add(recipientId);
    return "suppressed";
  }

  return { notify, applyVerifiedFeedback };
}

async function demonstrate() {
  const sent: Envelope[] = [];
  const notifications = createNotifications({
    async submit(message) {
      sent.push(message);
      return { receiptId: `receipt-${sent.length}` };
    },
  });

  const first: MatchNotification = {
    id: "notification-41",
    recipientId: "player-17",
    to: "player17@example.invalid",
    matchId: "match-204",
    template: "match-result-v1",
  };

  const initial = await notifications.notify(first);
  const duplicate = await notifications.notify(first);
  const feedback = notifications.applyVerifiedFeedback({
    receiptId: "receipt-1",
    kind: "permanent-recipient-failure",
  });
  const next = await notifications.notify({
    ...first,
    id: "notification-42",
    matchId: "match-205",
  });

  if (
    initial !== "submitted" ||
    duplicate !== "already-submitted" ||
    feedback !== "suppressed" ||
    next !== "suppressed" ||
    sent.length !== 1
  ) {
    throw new Error("Notification flow failed");
  }
}

void demonstrate().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Only one envelope reaches the fake transport.

A duplicate notification is ignored, and the next match notification is suppressed after feedback. Notice that the receipt maps back to the application's recipient identity; the feedback caller cannot choose an arbitrary player to suppress.

That mapping needs a real persistence strategy before deployment. The example assumes unique receipt IDs and sequential calls, and it loses its memory on restart. It also cannot prevent a concurrent submission that has already passed the suppression check. Those limits define the next work, rather than properties to assume from a successful demonstration.

## Keep bounce decisions separate from template decisions

Treat invalid-recipient suppression as an application policy with evidence. Persist the internal recipient ID, the affected address revision, the matched receipt, the classification, and the time of the decision. Keep the original feedback available under an appropriate retention policy so a support review can distinguish an invalid address from a delivery delay or a sender configuration issue.

Classify before suppressing.

Define the adapter's classification rules against the actual transport's documented event meanings. A temporary condition should enter a bounded retry policy; a confirmed permanent recipient failure should block subsequent notifications to the affected destination. An unknown event should remain unclassified until its meaning is established. Those are proposed policy choices, and the adapter needs tests that prove it makes them deliberately. For the illustrative player, imagine feedback for `receipt-1` arriving after the account has switched to a corrected address. Applying that old failure to the new destination would confuse the identity of the player with the identity of the failed mailbox. Preserve the original address revision in the receipt record, so the classifier can make a scoped decision even when account data has changed. This adds a lookup and another field to maintain. The benefit is a suppression decision that remains explainable after an address correction.

Address changes deserve their own rule. The small example suppresses a stable player ID, which is conservative but can keep blocking that player after they correct their address. In a deployed system, attach suppression to a recipient and address revision. Require the address-change workflow to establish eligibility for the replacement destination. Never clear the entire suppression record merely because another match event arrived.

Sender authentication is a separate concern. RFC 7208 defines SPF checks for whether a host is authorized to use a domain in the SMTP MAIL FROM or HELO identity. SPF is not evidence that a player address is valid, nor does its result authenticate an incoming feedback request. Keep sender-domain configuration, feedback authentication, and recipient suppression as distinct responsibilities. Otherwise, a sender-side problem can become a growing list of incorrectly blocked players.

## Ship a narrow slice with visible failure states

Start with one event. Use one revision.

Keep a fixture for its exact subject and body. Test the duplicate event, matched permanent failure, unmatched receipt, and a subsequent match event for the suppressed recipient. Add a throwing transport and confirm the notification is not recorded as submitted. Then test the ingress adapter independently: untrusted feedback must not reach `applyVerifiedFeedback`, and valid feedback must resolve to the intended submission.

The production worker needs an explicit state for uncertain submission. If the remote system accepts an email but the connection ends before the worker stores its receipt, automatically retrying may send twice.

A local `Set` cannot solve that.

Use a transport idempotency contract where one is available and documented, or keep the attempt unresolved for reconciliation. Make that trade-off visible: automatic retries buy speed, while unresolved attempts buy time to establish what happened.

Measure the decisions the application actually controls: jobs created, submissions accepted, suppression skips, unmatched feedback, and unresolved attempts. Submission acceptance should not be labeled delivery. Correlate records through notification and receipt IDs, and keep raw addresses and player-facing message bodies out of routine logs. A support view can expose restricted details when needed without turning every worker log into a copy of player communications.

Deploy the event capture and worker with compatible template revisions. If revision `v1` still appears in pending jobs, removing its renderer is a breaking change to that queue. Keep it until those jobs are drained or deliberately migrated. The weekly release should not silently rewrite yesterday's pending match notifications.

## What I would change at scale

Replace the in-memory stores with durable notification, attempt, receipt, and suppression records. Couple game-state changes to notification creation through a transactional outbox, and let workers claim jobs under a concurrency policy. Retain the final eligibility check, while acknowledging that feedback arriving after submission cannot retract that submission.

Keep template publishing separate from suppression authority. An editor may need to change match-result copy; that should not grant permission to reactivate invalid destinations. At higher volume, invest in feedback reconciliation and controlled replay before adding more template features. Revenue per hour still matters: own the decisions that affect players, outsource the undifferentiated transport, and ship weekly with a smaller, inspectable notification boundary.

## References

- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
