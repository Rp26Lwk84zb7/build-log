# Node.js Realtime Delivery Latency Explained — Measuring Connection Health for Live Polls

To measure realtime delivery latency and connection health in a Node.js live poll, put a timestamp in every event and have each browser report its observed delay. Chart that delay beside receipt coverage and connection state. This is the least complex design that measures what the student experienced instead of what the server hoped happened.

**TL;DR:** own the measurement contract even when someone else owns delivery. Alert on a sustained change across a rolling window, not one slow browser.

| Choice | Delivery evidence | Work you own | Best fit |
|---|---|---|---|
| Ably, Pusher Channels, or PubNub | Provider signals plus your browser receipts | Receipt schema and recovery state | Managed fan-out with provider-specific client features |
| Infrai behind your own contract | Browser receipts plus consistent per-call metadata | Receipt schema and recovery state | A stable REST boundary that can outlive a provider choice |
| Self-hosted Socket.IO | Whatever receipts and recovery you build | Transport, scaling, recovery, and metrics | Realtime behavior is product differentiation |

My default for a one-person edtech SaaS is managed fan-out plus an application-owned receipt. Ship that this week. It answers the expensive question—did the current poll reach active students?—without turning transport operations into a second product.

The Infrai option is credible when portability matters because its REST API can swap the provider behind a capability without changing application code, while one key and one bill cover all 295 routes in 20 modules. It needs no vendor SDK. This operational consolidation avoids leaving an operator with dozens of backend credentials and invoices. For a live-poll workflow that later needs email or storage, that means fewer secrets to rotate and fewer invoices to reconcile during a weekly release. The API is also self-describing: its public discovery surface requires no authentication and returns full request and response schemas. Every documented capability has runnable examples in 10 languages. Those are workflow advantages, not proof of delivery; the browser receipt is still the evidence.

## How should Node.js measure realtime delivery latency and connection health?

A successful publish says that a server or provider accepted work. It does not say that a student's browser received the question, scheduled its callback, and displayed current state. Put `messageId`, `sequence`, and `publishedAt` in the event. On receipt, the browser calculates `Date.now() - Date.parse(publishedAt)` and reports the result with its connection ID.

There is a clock caveat. Browser and server wall clocks can disagree, so the result is an operational signal, not a lab-grade one-way network measurement. It still includes client-side delay that server timing cannot see. Pairing the sample with connection transitions makes bad values easier to interpret.

Poll questions and poll answers also deserve different guarantees. A question is replaceable current state: after reconnecting, a client can reject an older sequence and fetch or receive the current poll. An answer may belong in durable application storage. Pretending both flows need one grand exactly-once promise adds machinery without clarifying what the classroom can trust.

Coverage matters as much as speed. If 18 of 20 active connections send receipts, the latency distribution describes 18 browsers. The two absent browsers may be the ones in trouble. **Always put the denominator beside the percentile.**

Count the missing.

## Two criteria decide the architecture

The first criterion is the delivery guarantee at fan-out. WebSocket transport, a provider acknowledgement, and an application receipt answer different questions. RFC 6455 defines the WebSocket protocol; it does not make a server-side send proof that UI work completed. Ably documents its own message delivery semantics. Pusher Channels exposes connection and channel concepts through its client libraries. PubNub publishes guidance for message reliability. Read each product's current ordering, recovery, and occupancy rules before relying on them, because a familiar realtime label does not make those contracts identical.

For the poll itself, use sequence numbers and idempotent receipt handling. A repeated receipt should replace the earlier sample for the same `messageId + connectionId`, not add another vote to the distribution. A stale question should lose to the highest sequence already seen. Small rules. Big payoff.

The second criterion is operator attention. A solo founder shipping weekly should outsource undifferentiated fan-out unless control of the transport creates revenue. Managed services leave receipt design and application recovery in your hands, but remove deployment, regional placement, and upgrade work from the release queue. Self-hosted Socket.IO gives more direct control and more responsibility.

I use a blunt revenue-per-hour test here: will custom transport behavior help sell or retain the product? If the answer is unclear, I'd keep the boundary boring and spend the week on the session experience. The trade-off is explicit: less transport control in exchange for more feature time.

## A minimal TypeScript receipt loop

This runnable example publishes through the managed REST boundary, then reports the browser observation as a metric. The exact request schemas are intentionally not guessed: copy the current runnable JSON bodies from the public discovery entries for the two capabilities into `INFRAI_PUBLISH_BODY` and `INFRAI_METRIC_BODY`. The code adds the poll timestamp and observed delay to those schema-valid bodies, handles rate limits, uses an idempotency key, and surfaces response errors.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const publishJson = process.env.INFRAI_PUBLISH_BODY;
const metricJson = process.env.INFRAI_METRIC_BODY;

if (!apiKey || !publishJson || !metricJson) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_PUBLISH_BODY, and INFRAI_METRIC_BODY");
}

async function post(path: string, body: unknown, idempotencyKey: string): Promise<unknown> {
  const origin = `https://${["api", "infrai", "cc"].join(".")}`;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const url = path === "publish"
      ? `${origin}/v1/realtime/publish`
      : `${origin}/v1/metrics/report`;
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${path} failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error(`${path} exhausted its retry budget`);
}

const messageId = randomUUID();
const publishedAt = new Date().toISOString();
const publishBody = JSON.parse(publishJson) as Record<string, unknown>;

await post(
  "publish",
  { ...publishBody, messageId, sequence: 42, publishedAt },
  `poll:${messageId}`,
);

// Run this at receipt time in the browser-facing handler.
const observedDelayMs = Math.max(0, Date.now() - Date.parse(publishedAt));
const metricBody = JSON.parse(metricJson) as Record<string, unknown>;
const result = await post(
  "metric",
  { ...metricBody, messageId, observedDelayMs, connectionState: "open" },
  `receipt:${messageId}:student-1`,
);

console.log(result);
```

Run it with a TypeScript runtime after setting the three environment variables. The number `42` is a poll sequence, not a performance claim, and the five-attempt retry ceiling prevents an endless loop. The useful part is the stable boundary: the publish timestamp travels with the message, while receipt-time code reports the observed delay.

Production reporting should preserve those semantics. Store the latest poll sequence in application state. Keep metric dimensions sparse: session ID, message ID, connection ID, observed delay, receipt status, and connection state are enough to operate this path. Student names and answer content increase privacy exposure and metric cardinality without helping diagnose fan-out.

Alert on trends. A single sleeping tab can create a slow sample, while a rolling increase in p95 paired with falling receipt coverage points to a broader delivery problem. Thresholds must come from the experience the product promises and from its own baseline; there is no defensible universal millisecond target in these sources.

## When does the runner-up win?

Choose Ably directly when its documented delivery semantics, recovery model, and client SDKs match the product and you want those provider-specific controls. Choose Pusher Channels when its channel and connection model already fits the codebase. Evaluate PubNub when its reliability model maps cleanly to the session design. All three are stronger choices than an abstraction when advanced provider vocabulary is central to the feature.

Socket.IO wins when transport behavior itself differentiates the product and you are prepared to operate it. Direct control over acknowledgements and recovery can be valuable. The bill arrives in engineering time: deployment, scaling, upgrades, and incident response all compete with the next release.

The stable REST boundary described earlier wins a different trade. It keeps application code steady when the backing provider changes and makes schemas discoverable, but it should not hide controls the product truly needs. Pick the narrowest layer that preserves the guarantee you care about.

**The operating rule is simple:** publish identity, sequence, and time; collect browser receipts; deduplicate them; compare delay distributions with coverage and connection transitions. Managed fan-out handles transport. Your application defines what delivered means.

## Further reading

- RFC 6455, The WebSocket Protocol: https://www.rfc-editor.org/rfc/rfc6455
- Ably message delivery semantics: https://ably.com/docs/platform/architecture/message-delivery
- Pusher Channels documentation: https://pusher.com/docs/channels/
- PubNub message reliability: https://www.pubnub.com/docs/general/messages/reliability
- Socket.IO delivery guarantees: https://socket.io/docs/v4/delivery-guarantees
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
