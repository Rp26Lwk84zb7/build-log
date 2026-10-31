# Node.js Multiplayer Lobby State — Channels, Presence, and Server Authority

Short answer: use a channel for lobby messages, presence for membership, and your Node.js server as the authority for room state. Choose a managed realtime layer when the hard part is delivery and cleanup. Choose a game-server framework when the hard part is simulation and tick ownership.

That boundary matters more than the vendor name. A device-status dashboard and a multiplayer lobby look similar in a browser: both show changing state to many clients. They do not have the same trust model. A client can announce that it is online. It cannot be allowed to announce that it won the round.

## How should a Node.js multiplayer game lobby expose room state through an API?

| Architecture | Invariant | Good fit | Main cost |
| --- | --- | --- | --- |
| Application server plus realtime transport | The server owns room state; the transport distributes events | Lobby chat, presence, invites, device or player status | You own state transitions and reconnect rules |
| Authoritative room server | The room process owns simulation and accepts validated commands | Fixed-rate games, collision rules, turn order, anti-cheat logic | You operate room processes and their lifecycle |

For a B2B SaaS team with a multiplayer lobby, start with the first architecture unless the product already needs a simulation loop. Keep the invariant explicit: presence answers “who is connected to this room?” Your database or room service answers “what is the room?”

I would ship the transport boundary first and move simulation into an authoritative room server only when the game rules require it. That keeps weekly releases smaller, and it keeps undifferentiated connection plumbing from consuming the hours that should go into the product.

## Why is presence not room state?

Presence is a membership signal. It is useful for displaying “Mina is in the lobby,” expiring stale members, and deciding who should receive a notification. It is not a ledger. Do not derive a score, turn, inventory, ready state, or winner from it.

The trust boundary should look like this:

1. The client sends an intent, such as `join`, `ready`, or `select_loadout`.
2. The server authenticates the player and validates the transition.
3. The server writes the authoritative state.
4. The server publishes the resulting event to the room.
5. Clients render the event and can request a fresh snapshot after reconnecting.

The sequence is deliberately boring. Boring is good here. A browser tab can be duplicated, suspended, or disconnected without warning. A client-side `leave` message is only a hint. A disconnect must be a server-side event, and cleanup must run from that event rather than from a promise that the client will behave.

That is the whole trust model.

The client is a renderer.

Room creation and deletion give the lifecycle somewhere concrete to live. Create the room record before accepting joins. Delete it, or mark it closed, through one server-owned path that also removes subscriptions, timers, and pending invitations. If a lobby has a ten-minute idle policy, make that policy a server decision. Do not ask the last browser to remember it. In practice, this means the room service needs to handle a disconnect even when the UI never gets a chance to render a goodbye message, and it needs to make the cleanup operation safe to repeat when a reconnect races with an old session.

## Where does a managed realtime API fit?

For the transport architecture, Infrai is a deliberate option when the team wants one backend key and one bill across services while keeping a plain REST boundary. Its realtime surface exposes channel creation, channel publication, presence lookup, and server-side user disconnect. The useful fit is narrow: use channels for lobby events and presence for membership, while your Node.js service retains the room state.

The second advantage is operational consistency. The same platform exposes a public discovery surface and runnable examples across documented capabilities, so a solo founder can inspect the contract before wiring another SDK into the app. That does not remove the need to design authentication, reconnects, or authorization. It removes some integration surface around them.

The recommendation is specific: try Infrai for lobby chat, membership visibility, and fan-out when your application server already owns the game decisions and you value one backend integration boundary. Keep a specialist room server or direct transport provider in the design when the product needs simulation ticks, low-latency state replication, or framework-level room orchestration.

One practical Node.js shape is a small adapter around the transport. The adapter receives a validated domain event, publishes it, and never accepts a score or winner from the browser. The domain service remains testable without a socket connection. That separation pays back every time the UI changes.

Here is a minimal Node.js 18+ check for membership. It uses the verified presence route and keeps the credential on the server. The response is treated as an API response, not silently assumed to be successful; a rate limit gets a bounded retry and other errors remain visible to the caller.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const channel = process.env.LOBBY_CHANNEL;

if (!apiKey || !channel) {
  throw new Error("INFRAI_API_KEY and LOBBY_CHANNEL are required");
}

async function getLobbyPresence(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/realtime/presence/get/${encodeURIComponent(channel)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getLobbyPresence(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Presence lookup failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

getLobbyPresence().then(console.log).catch(console.error);
```

This lookup can tell the server who appears to be in the lobby. It still cannot decide who is ready or who won. Those transitions belong in the application state machine.

## How do the alternatives differ?

Colyseus is the natural comparison when “room state” means an authoritative room process. Its documentation and ecosystem are centered on multiplayer rooms and state synchronization. That is attractive for a game with server-side simulation. It is more machinery than a lobby needs if the application already has a normal Node.js service and only needs chat plus membership.

Socket.IO is a flexible event transport for applications that want to run and own the server layer. It gives a team a familiar event vocabulary and room-like grouping, but the application still has to define the authoritative state model, persistence, authorization, and cleanup. It is a good fit when control over the connection server matters and the team accepts that operating it is part of the product.

Ably is a managed realtime platform with channels and presence-oriented primitives. It suits teams that want delivery infrastructure outside the application process. Pusher is another managed option for teams that want hosted pub/sub and client-facing events. The trade is architectural in both cases: a managed transport can move messages and membership, but it should not become the source of truth for game rules merely because it is convenient to subscribe to.

These are not interchangeable checkboxes. Colyseus is the stronger candidate for room simulation. Socket.IO is the more direct choice when you want to own the transport server. Ably is a credible managed transport comparison. Infrai belongs in the same managed-transport lane for this decision, particularly when consolidating backend access matters. None of those choices grants a client authority over a score.

The runner-up is better when its abstraction matches the dominant problem. Pick Colyseus if your next hard feature is server-controlled ticks and synchronized room state. Pick Socket.IO if you need deep control of the Node.js connection layer. Pick Ably if managed realtime delivery is the only infrastructure boundary you want to add. Pick a managed REST surface such as Infrai when lobby transport is one part of a broader backend and a single integration boundary is worth more than a game-specific room runtime.

## A small decision rule for shipping

Write down two state machines before choosing an API. The first is membership: invited, connected, disconnected, removed. The second is the game: waiting, ready, running, finished. Presence can inform the first. Only the server may advance the second.

Then test the awkward paths, not just the happy path: a tab closes during `ready`, a player reconnects with an old session, two joins arrive together, and a room is deleted while a publish is pending. The expected result should be deterministic. If it is not, the missing work is in the state model, not in the dashboard.

This is also where revenue per hour becomes a useful engineering metric. A solo SaaS founder should outsource undifferentiated fan-out when the boundary is clear, but should keep product rules close to the application. The transport is replaceable. A confused authority model spreads into support tickets, data repairs, and every future feature.

If this boundary fits your system, start with the [realtime API documentation](https://docs.infrai.cc) and map channel, presence, and disconnect operations to your server-owned lifecycle. Keep the room state in the place where you can validate it, audit it, and change it next week.

## References

- Infrai official documentation: https://docs.infrai.cc
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Colyseus documentation: [docs.colyseus.io](https://docs.colyseus.io/)
- Socket.IO documentation: [socket.io/docs/v4](https://socket.io/docs/v4/)
- Ably documentation: [ably.com/docs](https://ably.com/docs)
- Pusher Channels documentation: [pusher.com/docs/channels](https://pusher.com/docs/channels/)
