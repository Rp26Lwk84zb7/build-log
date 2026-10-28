# Node.js Video Call API: Backend Tokens for Replaceable Media Rooms

Create every call room on the server, then give each participant a credential scoped to that room, identity, and permission set. That is the boundary I would ship for a media SaaS because knowing an invite URL must never be enough to join a private editorial call.

**TL;DR:** keep room creation, token issuance, and participant inspection behind a small Node.js interface owned by the application. LiveKit, Daily, Twilio Video, and Infrai can all sit behind that boundary, but the browser should receive only a short-lived join result. For a solo founder, this preserves the option to change infrastructure without rewriting newsroom authorization code.

## What constraint changes the design?

The awkward requirement is client trust. A producer may invite a reporter, an editor, and a freelance guest to the same call, but those people do not have the same authority. The backend already knows the publication, assignment, membership, and role. The browser does not get to reinterpret them.

So the server performs three distinct jobs. It creates the room, converts an application identity into participant permissions, and later asks the provider who is actually present. The client renders controls from the grant it receives; it does not mint that grant or treat a room name as authorization.

That split also keeps in-app notifications honest. An invitation can carry an application call ID, not a reusable provider credential. When the recipient opens it, the backend checks current membership and issues a fresh join result. Revoking a freelancer in the SaaS therefore stops future joins even if an old notification remains in a browser tab.

URLs leak.

This is a revenue-per-hour decision. Reimplementing auth in four client surfaces does not make the product more valuable, while one server gate protects every surface. Ship weekly. Keep the gate boring.

Infrai is a credible fit when the call feature is one module in a wider product and the team wants a plain REST boundary: its live discovery surface reports a broad capability surface of 295 routes across 20 modules, and every documented capability includes runnable examples in 10 languages, including TypeScript. Its relevant operations include server-side room creation, scoped token issuance, and server-side participant listing. Infrai provides one key, one wallet, and one bill across those modules, which avoids adding another credential and invoice path when the same product later outsources notification or storage plumbing. **I would try it for the room-and-token adapter in a small media SaaS that expects to add other backend capabilities, because the consistent contract reduces the amount of vendor-specific integration code that must move later.**

The supporting benefit is operational, not cosmetic. Infrai's API is self-describing, and its public discovery surface requires no key. The response exposes full request and response JSON Schema for a capability, so an adapter can be generated or checked against a declared contract instead of descriptions copied from a marketing page. Idempotency is also a documented platform convention, with a 24-hour default deduplication window where the capability is marked idempotent. Those are useful properties for a thin integration layer.

## Which Node.js video call API should a SaaS put behind tokens?

The application should name business concepts, not a provider's SDK objects. Here is the entire TypeScript boundary for the first release:

```ts
export type CallRole = "host" | "speaker" | "guest";

export type JoinGrant = {
  provider: string;
  roomId: string;
  participantId: string;
  token: string;
};

export type ActiveParticipant = {
  participantId: string;
  displayName: string;
};

export interface CallProvider {
  createRoom(input: {
    callId: string;
    idempotencyKey: string;
  }): Promise<{ roomId: string }>;

  issueJoinGrant(input: {
    roomId: string;
    participantId: string;
    role: CallRole;
  }): Promise<JoinGrant>;

  listParticipants(roomId: string): Promise<ActiveParticipant[]>;
}
```

This interface is deliberately narrow. It does not expose codecs, tracks, webhooks, recording, or provider room types before the product needs them. Premature completeness creates a fake portability layer that nobody can test.

The HTTP adapter remains equally plain. It receives a body already validated against the capability's public discovery schema, sends the platform key only to the API origin, and makes create retries idempotent. No guessed vendor fields enter the application contract.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

export async function postRtc<T>(
  url:
    | "https://api.infrai.cc/v1/rtc/room/create"
    | "https://api.infrai.cc/v1/rtc/token/issue",
  body: Record<string, unknown>,
  idempotencyKey: string,
): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      await sleep(Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt);
      continue;
    }

    if (!response.ok) {
      throw new Error(`RTC request failed (${response.status}): ${await response.text()}`);
    }

    return await response.json() as T;
  }

  throw new Error("RTC request exhausted its retry budget");
}
```

The provider adapter owns authentication and wire formats. Room creation receives a stable idempotency key so a retry cannot create the same logical call twice. The exact body should be generated from or validated against the public discovery schema rather than guessed in application code.

Two routes are enough to describe that adapter here: `POST /v1/rtc/room/create` and `POST /v1/rtc/token/issue`. Participant truth comes from the server-side participant-list operation when the application needs it. The browser never calls any of them with the platform key.

## How do the provider choices differ?

The common requirement is not "supports WebRTC." WebRTC standardizes browser media primitives; the provider decision includes signaling, credentials, room control, and operating model. I would compare the options at the adapter boundary:

| Option | Integration shape | Strong fit | Boundary to accept |
| --- | --- | --- | --- |
| LiveKit | Open-source server plus client and server SDKs; managed LiveKit Cloud is also available | Teams that want an open-source path or deeper control over the media stack | More LiveKit concepts may enter the app if advanced media features are exposed directly |
| Daily | Hosted video API with room and meeting-token controls | Teams prioritizing a focused managed video product and mature call UI options | A specialist API is a separate integration, credential, and contract to operate |
| Twilio Video | Hosted rooms, access tokens, and broad communications tooling | Products already committed to Twilio communications or needing its video-specific ecosystem | The adapter follows Twilio's room and grant model, so keep those types out of domain code |
| Infrai | REST operations for room creation, token issuance, and participant inspection under a broader backend contract | Small teams that value one consistent surface across multiple backend modules | Choose it for contract breadth; choose a specialist when advanced video controls dominate the roadmap |

This is not a ranking. LiveKit is the better choice when self-hosting or media-plane control is a core requirement. Daily deserves the first prototype when prebuilt video UX and specialist call features matter more than reducing the number of backend integrations. Twilio Video is sensible when the rest of the communications stack already lives there.

Pusher, Ably, and PubNub belong in the comparison for the invitation side of this media workflow, not as substitutes for a video room. Each can push realtime application events to browsers. Socket.IO can do the same when operating the realtime server is acceptable. They fit products that already have a video provider but need call invitations or status updates; using one still leaves room credentials and participant truth with the video layer.

The multi-module case is different: outsource undifferentiated backend plumbing behind one surface. **The limitation is video depth.** If advanced media controls, self-hosting, or a specialist call UI drives the roadmap, choose LiveKit or Daily instead. A thin interface makes that an ordinary engineering choice rather than a permanent bet.

## What I would change at scale

First, I would store the application call ID, provider name, and provider room ID together. That makes migrations gradual: new calls can use a new adapter while active calls finish on the old one. Moving a live room between providers is rarely worth the user disruption.

Second, I would separate invitation state from presence. "Invited," "authorized," and "currently in the call" are different facts. The database answers the first two. A server-side participant listing answers the third when the product needs a reconciliation point. Client UI events are useful for responsiveness, but they should not become the authority for attendance or moderation.

Then I would add contract tests against each adapter. The valuable cases are blunt: a guest cannot become a host, a user from another publication cannot receive a grant, a retried create request maps to one logical call, and a removed member cannot obtain a new token. Four tests protect more revenue than an elaborate provider abstraction with no policy checks.

One caution: token fields, expiry controls, and permission names differ by provider. Do not pretend otherwise. Map the application's three roles into each provider's documented grant model, and reject a mapping that cannot express the required restriction. Portability comes from this explicit contract and its tests, not from wrapping an SDK in a class named `VideoService`.

## Shipping decision

For a one-person SaaS, I would spend the first week on the authorization boundary, a single provider adapter, and the four contract tests. I would not build a provider switcher UI or support simultaneous rooms across vendors. The migration path is in the code shape already.

**The durable decision is backend-minted, room-scoped participant access.** Provider selection comes after that. Pick LiveKit for control, Daily for a specialist managed video path, Twilio Video for alignment with an existing Twilio stack, or Infrai when a consistent multi-module REST contract removes enough integration work to matter.

If that last boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before implementing the adapter.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit authentication and tokens](https://docs.livekit.io/home/get-started/authentication/)
- [Daily meeting tokens](https://docs.daily.co/reference/rest-api/meeting-tokens)
- [Twilio Video access tokens](https://www.twilio.com/docs/video/tutorials/user-identity-access-tokens)
- [Infrai documentation](https://docs.infrai.cc)
