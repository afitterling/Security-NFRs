# AUTH-005 — Federated sign-in (Apple & future IdPs)

- **Status:** Adopted
- **Group:** Authentication
- **Applies to:** Every app offering Sign in with Apple or another external IdP.
- **Last updated:** 2026-06-16

## Requirement

1. An identity token from an external IdP **MUST** be fully verified before any
   account is created or logged in: signature against the IdP's JWKS, plus
   `iss`, `aud`, and `exp`. An unverifiable token **MUST** be rejected (401).
2. When the client performs a native sign-in and posts the identity token, the
   server **MUST** verify a **nonce** it bound to the request, to stop replay of
   a captured identity token.
3. The authorization-code (browser) leg **MUST** use PKCE (S256) and a
   CSRF-protected, integrity-checked `state` (see
   [AUTH-002](AUTH-002-no-tokens-in-urls.md)).
4. When the IdP omits the email on subsequent logins, the account **MUST** be
   keyed on the stable subject (`sub`), not on a possibly-absent email.
5. The session/tokens issued after federation are subject to all of
   [AUTH-002](AUTH-002-no-tokens-in-urls.md) and
   [AUTH-003](AUTH-003-session-token-lifecycle.md).

## Rationale

A federated login is only as strong as its token verification. Missing nonce or
`aud`/`iss` checks turn "Sign in with Apple" into a replayable bypass; PKCE +
bound `state` protect the browser leg.

## Acceptance criteria

- [ ] A token with a wrong `aud`/`iss`/signature is rejected.
- [ ] A previously-captured identity token replays unsuccessfully (nonce mismatch).
- [ ] The browser callback rejects a forged/cross-browser `state`.
- [ ] Second login with no email still resolves to the same account via `sub`.

## Implementation notes

- **OpenCycle / OpenOutdoor:** Apple federates through Cognito; `/apple`→`/callback` now use signed + cookie-bound `state` and S256 PKCE.
- **Priorize:** backend verifies the Apple identity token (sig/iss/aud/exp) correctly. **Gap:** native-post path does not yet verify a client nonce (replayable) — tracked.
