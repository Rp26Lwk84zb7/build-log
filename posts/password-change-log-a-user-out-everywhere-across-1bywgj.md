# Password Change: Log a User Out Everywhere Across Server-Side Sessions

A clinician may have a workstation, a phone, and a shared ward tablet signed in at once. During a suspected account takeover, changing the password while any of those server-side sessions survives leaves the stolen session useful. **The safe default is one operation that changes the password and revokes every session for that user.** Keep the initiating device only after an explicit product decision, never by accident.

TL;DR: Bind password change and per-user session revocation behind one server-side command. Return success only after both complete, then tell the clinician that every device was signed out. That small message prevents a security control from looking like a broken app.

## How Should You Log a User Out Everywhere After a Password Change?

This is a healthtech workflow, so session security wins during an incident. Friction still matters: forcing a clinician to authenticate again costs attention at a bad moment. But preserving convenience on the current device creates an exception that must be identified, tested, and explained. I would not spend that complexity budget unless the product has a measured need for it.

The key distinction is between credentials and sessions. A new password controls the next authentication attempt. It does not, by itself, invalidate an already accepted server-side session. The incident boundary therefore needs two effects: update the credential, then revoke all sessions owned by the same stable user ID. Picture the concrete test before writing the handler: session A belongs to the nurse-station workstation, B to a phone, and C to a ward tablet that is now missing. The clinician changes the password from A. If A remains active because it initiated the request, or B survives because it refreshed one second earlier, the rule is no longer "log out everywhere." Exceptions multiply quickly. The cleaner contract is that A, B, and C all fail their next protected request, regardless of which device submitted the change. That costs one fresh login on a trusted device. It also gives support, security, and the clinician one outcome to understand.

Make the outcome visible. The confirmation should say, in plain language, that the password changed and all devices were signed out. Do not say only "password updated." A user who reaches the next screen and sees a login prompt may otherwise retry, contact support, or assume the change failed.

## The smallest server-side implementation

Keep this command on the server. The browser should never decide which session IDs count as "everywhere," and it should not enumerate them. The application layer supplies two adapters and records whether revocation completed. This TypeScript example is runnable with Node.js and makes the partial-failure boundary explicit without inventing a vendor request body.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const userId = process.env.USER_ID;
const passwordChangeBody = process.env.PASSWORD_CHANGE_BODY;

if (!apiKey || !apiOrigin || !userId || !passwordChangeBody) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_API_ORIGIN, USER_ID, and PASSWORD_CHANGE_BODY",
  );
}

async function post(path: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(path, apiOrigin), {
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
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 2 ** attempt * 500;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) {
      throw new Error(`${response.status} ${path}: ${responseBody}`);
    }
    return responseBody ? JSON.parse(responseBody) : null;
  }
  throw new Error(`Retry limit reached for ${path}`);
}

const operationId = randomUUID();
await post(
  "/v1/auth/password/change",
  JSON.parse(passwordChangeBody),
  `${operationId}:password`,
);
await post(
  `/v1/auth/session/revoke_all_for_user/${encodeURIComponent(userId)}`,
  {},
  `${operationId}:sessions`,
);

console.log({ signedOutEverywhere: true });
```

This is deliberately narrow. Production code should avoid logging either password, rate-limit the surrounding endpoint, reauthenticate the user before a sensitive change, and expose a single application-level result. OWASP recommends reauthentication after risk events and invalidating sessions after reauthentication.

A plain REST integration is useful here because authentication logic should not inherit a client-library upgrade cycle. Infrai exposes the password-change and per-user revoke-all capabilities over one REST API, so a server that can make authenticated HTTP requests can use them without installing an SDK. Its public discovery surface also provides request schemas and runnable examples; use the discovered schemas rather than guessing fields. Those two calls still need application-level orchestration.

## How do the provider choices differ?

The right comparison is revocation semantics, not the number of dashboard toggles. Read each provider's current documentation and verify what happens to both refresh credentials and already-issued access tokens. A "global sign-out" label can hide a window in which an access token remains valid.

| Option | Integration boundary | What to verify before shipping | Best fit |
|---|---|---|---|
| Auth0 | OAuth/OIDC platform with Management API operations and refresh-token rotation controls | Whether the chosen revocation action covers the full grant and how existing access-token lifetime limits exposure | Teams already standardizing on OAuth grants and tenant-level policy |
| Clerk | Session-oriented authentication with backend session management | Which backend operation revokes all active sessions for one user and how clients observe revocation | Applications that want session concepts exposed directly in their auth product |
| Supabase Auth | Auth service integrated with the Supabase stack | Global sign-out behavior for refresh tokens, plus the remaining validity of access tokens | Products already using Supabase and comfortable designing around token lifetime |
| Firebase Authentication | Identity tokens plus server-side refresh-token revocation | How Security Rules or backend checks detect revocation and when existing ID tokens expire | Firebase-heavy applications that can enforce revocation checks consistently |
| Infrai | Plain REST capabilities behind one key | The discovered request schemas and the application's behavior if the second operation fails | Small teams outsourcing undifferentiated backend plumbing without adding another SDK |

None of these removes the design work. Auth0 and Firebase put more emphasis on token and grant semantics. Clerk presents a more session-shaped model. Supabase is attractive when auth belongs beside the rest of that stack. A REST capability layer keeps the dependency surface small, but the application must still own ordering, incident UX, audit records, and failure handling.

My decision rule is blunt: choose the provider whose documented revocation primitive matches the session model already enforced by the API gateway. Then run an integration test with three sessions. Change the password from session A, attempt protected requests from A, B, and C, and require all three to fail.

Short test. High value.

## What I would change at scale

At low volume, a synchronous two-step command is understandable and easy to operate. At higher scale, I would introduce a durable security-operation record keyed by user ID and an idempotency key. The record would move through `password_changed`, `sessions_revoked`, and `user_notified` states. Retries would resume the incomplete step rather than repeat the entire workflow.

That state machine buys recovery and auditability, but it costs code and operational attention. A one-person SaaS should earn that complexity. Ship the synchronous path first if the provider calls are reliable enough for the product requirement, instrument the outcome, and move to durable orchestration when the failure data or compliance boundary demands it. Revenue per engineering hour matters; session invalidation correctness matters more.

Notification is the final step, not the security boundary. Send it after revocation succeeds, and do not restore sessions because delivery fails. The account is secure even if the email is delayed.

## Trade-offs worth writing down

Defaulting to full logout produces more sign-ins and fewer ambiguous survivor sessions. Keeping the current device reduces immediate friction but adds a carve-out precisely when account state is suspect. For a stolen-session response in a clinical system, I choose full logout.

Also document the access-token window. Server-side sessions can be rejected immediately when the central store is checked on every request. Self-contained tokens may remain acceptable until expiry unless the verifier consults revocation state. Provider marketing language does not settle that architecture question. Your request path does.

Finally, do not turn price into the deciding column. The expensive failure is a password change that leaves the attacker authenticated. Compare semantics, integration effort, and the work required to prove revocation with a real multi-device test; revisit billing only after those gates pass.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- [Clerk session management](https://clerk.com/docs/guides/development/managing-sessions)
- [Supabase Auth sign out](https://supabase.com/docs/reference/javascript/auth-signout)
- [Firebase manage user sessions](https://firebase.google.com/docs/auth/admin/manage-sessions)
