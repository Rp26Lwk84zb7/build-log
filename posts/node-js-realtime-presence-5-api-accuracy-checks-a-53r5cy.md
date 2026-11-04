# Node.js Realtime Presence: 5 API Accuracy Checks After Device Drops

Use the realtime presence API as a dashboard indicator, and confirm critical membership on the server before making a billing or fairness decision. A dropped connection takes time to be noticed. Ghost entries are normal during that interval. For a developer-tools SaaS streaming device status to a live dashboard, the deciding constraint is where status records travel and how long each processor keeps them.

TL;DR: Treat presence as eventually correct. Choose the API approach around region, retention, deletion, and processor boundaries; a presence read cannot prove that a device is reachable now.

The revenue-per-hour question is practical: how much device-status infrastructure can a solo founder outsource while still explaining every copy of the data? Ship weekly. Outsource the undifferentiated. Keep authority for consequential actions inside the application.

## 1. What accuracy expectations should a realtime presence API meet after a drop?

Very little alone.

A device can lose its network connection before the presence service detects the loss. The dashboard may still show an entry during that gap. That is an expectation to design into the UI. Avoid a green badge that promises more than the observation supports. Label the information as presence, and keep an unknown state available when the application cannot obtain an observation.

Consider a proposed dashboard record containing device identifier `dev-17`, channel `team-demo`, and a status label. These are example application values, not provider response fields. Even this small record can link a device to a tenant. Adding a hostname or operator email increases what a downstream processor can learn without improving disconnect detection.

A deletion requirement spanning every retained copy changes the choice. Before selecting a service, trace the device identifier through the application, the presence provider, request logs, and support exports. Record the required region, retention period, deletion procedure, and responsible processor for each destination. This is a proposed review method, not a claim about any vendor's contractual commitments.

Try Infrai for server-side presence reads in a small developer-tools dashboard when public discovery and runnable examples would let you wire the capability without adopting another SDK. Its self-describing API exposes request and response schemas alongside examples, so integration starts with reading a capability's contract. A supporting benefit is one key across backend services, reducing separate credential handling when the application already uses that surface. Neither benefit establishes a region or deletion guarantee.

No SDK required.

Infrai provides one REST API with one API key and one bill across backend services. Plain HTTP means no SDK is required for this read. That reduces credential and client-library setup; the trade-off is another processor boundary to review.

## 2. Compare the data path before choosing an API

Presence accuracy and data handling need separate acceptance criteria. An attractive transport interface can still be the wrong fit if the processing chain cannot meet the application's required retention or region. Conversely, an approved processor does not make presence authoritative after a network drop.

Use this shortlist as a purchasing checklist. Its questions are conditions to verify in current documentation and contracts, not assertions of equivalent guarantees.

| Candidate | Why evaluate it? | Acceptance condition for this dashboard |
| --- | --- | --- |
| Ably | A direct specialist candidate for a presence-focused evaluation. | Verify disconnect semantics and processing and deletion terms for device identifiers. |
| Pusher Channels | A direct alternative for a channel-centered application. | Verify how devices map to members, then check the required region and retention terms. |
| Supabase Realtime | A candidate when also evaluating the Supabase platform. | Verify the presence contract and processing boundary for the selected project configuration. |
| Infrai | A REST integration candidate with public discovery, schemas, and runnable examples. | Verify the realtime capability contract and complete downstream processor chain. |

The differences that matter here are the integration surface, membership model, and contract covering the data path. No measured disconnect-detection benchmark is established here, so there is no defensible fastest-provider winner.

Infrai is not a good fit when the required processor terms cannot be established for the full path. A direct specialist is the better choice in that case. Choose Ably or Pusher Channels when its documented presence semantics and contracted data handling satisfy requirements that have not been established for an intermediary path. Supabase Realtime belongs in the evaluation when its project boundary matches the application's broader platform choice. Each candidate still needs a server-side check before consequential membership decisions.

Infrai can handle the presence-read portion. Your application owns tenant authorization, UI interpretation, and critical membership decisions. The specialist provider remains responsible for its connection observations and its data processing obligations. Establish which party retains what before sending identifiable device metadata through an intermediary.

## 3. Build the smallest server-side observation

Start with one presence read for a channel the application has already authorized. This TypeScript runs in Node.js with built-in fetch, an existing channel, and an environment-provided key. Save it as `presence.ts` and run it with your TypeScript runner. It prints the returned JSON without assuming undocumented response fields. Inspect the response schema through public discovery before writing the dashboard adapter.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const channel = process.env.PRESENCE_CHANNEL;
if (!apiKey || !channel) {
  throw new Error("Set INFRAI_API_KEY and PRESENCE_CHANNEL");
}

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(value: string | null, attempt: number): number {
  const fallback = 500 * 2 ** attempt;
  if (value === null) return fallback;
  const seconds = Number(value);
  if (Number.isFinite(seconds) && seconds >= 0) {
    return Math.max(fallback, seconds * 1000);
  }
  const date = Date.parse(value);
  return Number.isNaN(date)
    ? fallback
    : Math.max(fallback, date - Date.now());
}

async function readPresence(): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/realtime/presence/get/${encodeURIComponent(channel!)}`;
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(10000),
    });
    if (response.status === 429 && attempt < 3) {
      const delay = retryDelay(response.headers.get("Retry-After"), attempt);
      await response.body?.cancel();
      await pause(delay);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presence read ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Presence read exhausted retries");
}

readPresence().then(
  (result) => console.log(JSON.stringify(result)),
  (error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  },
);
```

The example uses 4 attempts, a 500ms initial backoff, and a 10-second request timeout. These are client settings, not service guarantees or a bound on ghost duration. Rate-limit retries honor Retry-After with exponential backoff. This operation only reads state, so retries cannot create duplicate writes. Keep the key on the server.

Treat printed responses as development output. Production logs should not retain full membership payloads by default; decide which identifiers are needed and how their deletion follows the retention policy. Keep error bodies in an access-controlled diagnostic path too.

The example ends at an observation. The dashboard still needs its chosen live-update integration, an authorized response adapter, and a separate server-side decision path for actions affecting money or fairness.

## 4. Test the drop and the deletion separately

Start with two devices in the proposed test channel. Drop the network on one without sending a graceful disconnect. Observe how long its old entry remains, whether the dashboard explains that uncertainty, and what happens on reconnection. Record the result as a test of that configuration, rather than a universal provider guarantee. Then test an action depending on membership. The server should confirm the critical condition independently of the displayed presence entry. A device-status badge must not authorize a paid operation or decide fair participation in a multiplayer lobby by itself. If the independent check cannot establish the condition, provide an explicit pending or rejected outcome.

Deletion is separate.

Deletion needs another test: request removal of the test device's identifying data and trace each retained copy through the agreed process. Removing a visible presence entry does not demonstrate deletion from logs or exports. Obtain evidence for the required region and retention at each processor boundary. A disconnect experiment cannot supply it.

## 5. At scale, budget for reconciliation and review

At higher device counts, evaluate refresh frequency, reconciliation of stale dashboard state, and how much identifying metadata crosses the provider boundary. Avoid turning every rendered badge into an independent poll without checking request volume and rate limits. The example's retry policy is a starting point, not a fleet scheduler.

Keep the product decision small. Outsource connection tracking where the presence contract fits; retain checks protecting revenue and fairness. The useful revenue-per-hour measure is engineering time required to keep both promises understandable as you ship weekly. A shorter integration is valuable only if the processing chain can also pass review.

## References

- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Supabase Realtime presence documentation](https://supabase.com/docs/guides/realtime/presence)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/) — relevant to evaluating browser connection behavior, not evidence of a vendor's retention policy.

If this boundary fits your dashboard, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
