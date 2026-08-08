# AUTH-002 — No tokens in URLs (one-time code + PKCE)

- **Status:** Adopted
- **Group:** Authentication
- **Applies to:** Every app whose native client obtains a session via a hosted
  (browser) sign-in that hands control back through a deep link.
- **Last updated:** 2026-06-16

## Requirement

1. Access, ID, and refresh tokens **MUST NOT** appear in a URL (query or
   fragment), a redirect `Location`, the browser history, or an interstitial
   page body. URLs leak via logs, the `Referer` header, and shared history.
2. The browser→app handoff **MUST** pass only a **single-use, short-lived code**
   (recommended TTL ≤ 2 min). The app exchanges it for tokens over a `POST` to a
   dedicated endpoint. The code **MUST** be deleted on first use (atomic redeem),
   so a replay yields nothing.
3. The exchange **MUST** be bound to the initiating client with **PKCE (S256)**
   per RFC 7636: the app sends `code_challenge` when opening the flow and proves
   possession of the `code_verifier` at exchange. An intercepted code without the
   verifier **MUST** be rejected. (RFC 8252, "OAuth for Native Apps".)
4. The redirect/deep-link target **MUST** be validated against a fixed allowlist
   of exact values — never an open `startsWith` on the app scheme.
5. The browser leg's `state` **MUST** be unguessable, integrity-protected, and
   cross-checked (cookie-bound) to defeat CSRF/login-fixation on the callback.

## Rationale

Tokens in URLs are the most common silent leak in mobile OAuth-style flows. A
one-time PKCE-bound code reduces an intercepted handoff to a useless artifact and
is the documented gold standard for native apps.

## Acceptance criteria

- [ ] After sign-in, no token string is present in any redirect URL or rendered page.
- [ ] Re-POSTing a consumed code returns 401.
- [ ] An exchange with a missing/incorrect `code_verifier` (when a challenge was bound) returns 401.
- [ ] `redirect_uri=https://evil.example` is rejected (400).
- [ ] A callback with a forged/cross-browser `state` is rejected.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `appHandoff()` mints a code (`authstore.ts`), `POST /api/auth/exchange` redeems it; S256 PKCE captured at `GET /login|/signup|/apple`, verified at exchange; `safeRedirect()` allowlist; signed + cookie-bound `state` on the Apple leg.
- **Emergency:** one-time login token (5-min, single-use, deleted on consume) for the app→web handoff; the return deep link carries no token. **Gap:** handoff token still travels in the URL and is not yet PKCE-bound (tracked).
- **Priorize:** review against this spec (tracked).
