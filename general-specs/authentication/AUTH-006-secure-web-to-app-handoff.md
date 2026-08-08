# AUTH-006 — Secure web→app token hand-in

- **Status:** Adopted
- **Group:** Authentication
- **Applies to:** Every app where a **web** sign-in must deliver a session/token
  to a **native** client (the app opens a hosted login and expects to come back
  signed in).
- **Last updated:** 2026-06-16

## Feature

After the user authenticates on the web (email/password, or federated), the web
side **MUST** hand the resulting tokens to the app over a **secure channel** —
never by exposing tokens to the device's URL handler, browser history, or page
content. This is a first-class feature of every hosted-login app, not an
implementation detail.

## Requirement

1. The web login **MUST** deliver tokens via the **one-time code + PKCE** exchange
   defined in [AUTH-002](AUTH-002-no-tokens-in-urls.md):
   - the app opens the web login with a `code_challenge` (S256) and an
     unguessable `state`;
   - on success the web returns **only a single-use, short-lived code** on the
     deep link (`<scheme>://auth?code=…&state=…`) — **no tokens**;
   - the app `POST`s `{ code, code_verifier }` to `/api/auth/exchange` over HTTPS
     and receives the tokens in the response body.
2. The code **MUST** be single-use (atomic redeem, deleted on first use), short-
   lived (≤ 2 min), and bound to the device via PKCE; an intercepted code without
   the verifier **MUST** fail.
3. The deep-link target **MUST** be on the pinned allowlist; `state` **MUST** be
   integrity-checked/cross-checked to prevent CSRF/login-fixation.
4. The tokens received **MUST** then be stored and managed per
   [AUTH-003](AUTH-003-session-token-lifecycle.md) (secure storage, short access,
   revocable refresh).
5. For app→web→app flows that only need to establish a **browser session** (not
   give the app tokens), the web **MUST** set a hardened session cookie and the
   return deep link **MUST** carry no token (only a status signal).

## Rationale

The web→app hand-in is exactly where tokens classically leak (URL params,
`Referer`, history, interstitial HTML). A one-time PKCE-bound code is the RFC 8252
gold standard and makes an intercepted handoff worthless. Making it an explicit,
named feature ensures every app implements the *secure* delivery, not a quick
token-in-URL shortcut.

## Acceptance criteria

- [ ] After web sign-in, the deep link back to the app contains a `code`, never a token.
- [ ] Exchanging the code returns tokens over HTTPS POST; replaying the code returns 401.
- [ ] An exchange with a wrong/missing `code_verifier` (when bound) returns 401.
- [ ] The app stores the received tokens in the secure store and can refresh/revoke them.
- [ ] Browser-session-only handoffs return no token in the deep link.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `appHandoff()` mints a code (`authstore.ts`); `POST /api/auth/exchange` redeems with S256 PKCE; app `auth.ts` generates the verifier, opens `/login?...&code_challenge=…`, exchanges the code. Web browser-session path (OpenCycle `redirect_uri=web`) sets cookies, no token on the link.
- **Emergency:** app→web handoff uses a one-time, single-use, 5-min login token that sets a server session cookie; the return deep link carries only `status=ok`. **Gap:** the handoff token still rides in the URL and is not PKCE-bound — promote to the code+PKCE pattern.
- **Priorize:** review the app handoff against this spec (tracked).
