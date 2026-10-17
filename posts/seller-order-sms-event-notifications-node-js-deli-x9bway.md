# Seller Order SMS Event Notifications: Node.js Delivery Status Polling with Template Control

TL;DR: Treat a seller's new-order text as a versioned business artifact, not a string embedded in an SMS client. Render it once, persist the exact result, and let a small transport adapter handle submission, delivery polling, cancellation, and bounded resends. Put country eligibility and rate policy before that adapter. This keeps EU and US decisions reviewable without tying the marketplace's language or order logic to a provider.

For a one-person SaaS, this boundary matters more than another messaging feature. Template edits happen often; transport swaps should be rare. The revenue-per-hour choice is to own the copy and state machine, then outsource carrier delivery as undifferentiated work. Ship the narrow path first.

## How should Node.js check SMS event notification delivery?

A new-order alert combines business facts with delivery mechanics. The business side knows the seller, order reference, publication, and management link. The transport side should know only a destination, rendered text, idempotency key, and policy snapshot. Mixing those concerns makes a copy edit an infrastructure change and makes a transport migration a content migration.

Store three things with each notification: the template version, the normalized input used to render it, and the rendered body. The version explains which rules produced the message. The input supports deterministic tests. The body is the immutable payload submitted on every attempt for that notification. A resend should not silently pick up tomorrow's wording.

Words are state.

There is a hard trade-off here. Persisting the rendered body uses more storage and may retain data longer than persisting a template key alone. In return, support can answer a concrete question: what did the system attempt to send? Set retention deliberately, restrict access, and avoid putting unnecessary order details in the text. A link to the authenticated order page usually carries less disclosure than a full line-item summary.

## The smallest useful implementation

The first release needs one queue consumer and one provider-neutral port. Keep the state vocabulary local even if a transport uses different names. `queued`, `submitted`, `delivered`, `failed`, and `canceled` are enough for the application workflow; preserve the raw remote status separately for diagnosis.

```ts
type Region = "EU" | "US";
type NoticeState = "queued" | "submitted" | "delivered" | "failed" | "canceled";

type OrderNotice = {
  id: string;
  orderId: string;
  sellerPhone: string;
  region: Region;
  templateVersion: "seller-order-v1";
  renderedBody: string;
  state: NoticeState;
  attempt: number;
  remoteId?: string;
  nextCheckAt?: Date;
  policyVersion: string;
};

interface SmsTransport {
  submit(input: {
    to: string;
    body: string;
    idempotencyKey: string;
  }): Promise<{ remoteId: string }>;
  getStatus(remoteId: string): Promise<{
    state: "pending" | "delivered" | "failed";
    rawState: string;
  }>;
  cancel(remoteId: string): Promise<{ accepted: boolean }>;
}

type SendPolicy = {
  enabled: boolean;
  maxAttempts: number;
  pollAfterMs: number;
};

function renderSellerOrder(input: {
  orderRef: string;
  publication: string;
  orderUrl: string;
}): string {
  return `New order ${input.orderRef} for ${input.publication}. Review: ${input.orderUrl}`;
}

async function submitNotice(
  notice: OrderNotice,
  policy: SendPolicy,
  sms: SmsTransport,
  now: Date,
): Promise<OrderNotice> {
  if (!policy.enabled || notice.state !== "queued") return notice;
  if (notice.attempt >= policy.maxAttempts) return { ...notice, state: "failed" };

  const result = await sms.submit({
    to: notice.sellerPhone,
    body: notice.renderedBody,
    idempotencyKey: `${notice.id}:${notice.attempt}`,
  });

  return {
    ...notice,
    state: "submitted",
    remoteId: result.remoteId,
    nextCheckAt: new Date(now.getTime() + policy.pollAfterMs),
  };
}
```

The `SendPolicy` values are configuration, not universal limits. For example, a test environment might use `maxAttempts: 2` and `pollAfterMs: 30_000` to exercise transitions quickly. Production values must come from the transport contract, the marketplace's risk tolerance, and a reviewed country policy. Do not present illustrative numbers as carrier or legal rules.

Create the notification in the same business workflow that accepts the order, but enqueue it through a durable boundary after the order commit. Use the notification ID plus attempt number as the idempotency key. A worker crash can then repeat the submission call without inventing a new logical attempt, provided the selected transport honors that key; if it does not, record submission intent before calling it and reconcile ambiguous outcomes before resending.

That last case is the trap. A timeout does not prove failure. It means the result is unknown. Poll first.

## Which delivery failures require a new attempt?

Polling should advance local state, not trigger open-ended retries. When the remote state is pending, schedule another check. When it is delivered, stop. When it is failed, classify the result before deciding whether a new attempt is allowed. Keep the remote status and timestamp alongside the normalized state so an operator can inspect lossy mappings.

A resend is a new delivery attempt for the same immutable notification. Increment `attempt`, clear the old remote identifier, and return the record to `queued` only after policy approves it. Never resend merely because a poll was late. Network delay, delayed status propagation, and worker downtime can all make a healthy submission look stale.

Cancellation is conditional. Cancel a queued job locally with no transport call. For a submitted job, ask the adapter to cancel, then record whether the request was accepted; do not rewrite a delivered notification as canceled. The UI should say that cancellation was requested when final delivery is still unsettled.

Use two rate controls. A destination bucket prevents repeated texts to one seller during an order burst. A region bucket caps aggregate submission pressure. The queue should defer work when either bucket is empty, without consuming an attempt. This separation avoids punishing a seller for marketplace-wide traffic and keeps regional policy visible.

Consider a publisher who lists ten placements and receives six orders close together. Six order events may deserve six durable notification records because each order must remain auditable, but they do not automatically deserve six immediate texts. The destination bucket can release the first alert and defer the rest; a later worker may combine only records whose approved template and business rules explicitly permit aggregation. Meanwhile, the region bucket can pause every destination without marking any order failed. If the first submission times out, its state remains `submitted` or `unknown` in the local model until reconciliation establishes an outcome. It does not rejoin the queue beside the other five. This is why rate limiting, retrying, and batching cannot share one counter: they answer different questions about pressure, uncertainty, and message meaning. The extra records cost storage and a little worker code. They save the much harder support conversation in which nobody can tell whether one seller received zero texts, one text, or seven.

Unknown stays unknown.

## Country guardrails belong before submission

`EU` and `US` are policy selectors, not complete legal models. Country is more precise than region, and destination prefixes alone may be insufficient for deciding a recipient's applicable policy. Resolve the policy from verified account and destination data, version it, and attach that version to the notification before enqueueing. If the data is missing or contradictory, hold the notification for review or use a separately approved fallback channel. Do not guess.

Stop there.

The guard should answer a small set of application questions: is this transactional purpose enabled, is this template version approved, is the destination eligible, what sending window applies, and which rate bucket governs submission? Legal and operational owners must supply those answers. The code's job is to enforce and audit them, not fabricate them.

Template ownership makes this review practical. A diff from `seller-order-v1` to `seller-order-v2` shows the exact copy change. Tests can assert that order references and authenticated links render correctly, prohibited fields stay absent, and every enabled country maps to an explicit policy. The transport adapter remains unchanged.

Email fallback deserves its own template and delivery record. Google's sender guidance documents authentication and sending practices for email; it does not turn an SMS consent decision into email permission. Keep channel eligibility separate even when both notices originate from the same order event.

## What I would change at scale

Start with polling because it is easy to reason about and test. At higher volume, accept authenticated status callbacks when the transport supports them, while retaining a slow reconciliation poll for records that remain unsettled. Both paths must call the same transition function. Otherwise, callback behavior and polling behavior drift.

I would also split template publication from application deployment. A reviewed template registry can emit an immutable version and checksum; the order service accepts only published versions. That adds process, so it is premature when one person owns every edit. Add it when review latency or change volume is the constraint, not because a diagram looks cleaner.

The operational dashboard needs counts by local state, policy version, country, template version, and attempt. Alert on age in `submitted`, not just raw failure count. A sudden absence of submissions can be as important as a spike in failures. Keep message bodies and phone numbers out of metric labels and routine logs.

For agent-driven order operations, expose a narrow tool such as `queue_seller_order_notice` with an explicit schema rather than giving a model direct access to the transport. The tool-use guidance in Further reading explains why detailed tool definitions and clear input schemas matter. The server must still perform authorization, template selection, country checks, and idempotency. A model request is input, not policy.

The final decision rule is plain: **own the words and workflow; rent the delivery edge**. This creates a little application code, but it buys stable tests, explainable resends, and country rules that can change without rewriting order copy. For a weekly shipping cadence, that is the useful boundary.

## Further reading

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- Anthropic, Tool use overview: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
