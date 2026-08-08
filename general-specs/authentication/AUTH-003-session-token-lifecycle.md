# AUTH-003 — Session & token lifecycle

- **Status:** Adopted
- **Group:** Authentication
- **Applies to:** Every app that issues sessions or bearer tokens.
- **Last updated:** 2026-06-16

## Requirement

1. Access tokens **MUST** be short-lived (≤ 1 hour recommended). Long-lived
   access **MUST** be obtained by refreshing, not by extending an access token.
2. **Refresh tokens MUST be limited to a ~1-week lifetime.** A leaked refresh
   token is a long-lived key; capping it at one week bounds the exposure window
   (vs. the multi-month defaults some providers ship with). Re-authentication
   after a week of inactivity is acceptable.
3. **Clients MUST refresh regularly** — proactively, shortly before the access
   token expires (e.g. refresh ~5 min early), not lazily on the first 401 — so a
   live session stays valid without holding a long access token. Where the
   provider supports **refresh-token rotation**, each refresh **SHOULD** issue a
   new refresh token and invalidate the previous one.
4. There **MUST** be a working **logout that revokes server-side** — clearing the
   client copy alone is not logout. Revocation **MUST** invalidate the refresh
   token (and, where supported, outstanding access tokens) so a stolen/old token
   stops working. Token revocation **MUST** be enabled on the provider.
5. Tokens stored on a device **MUST** be kept in the platform secure store
   (iOS Keychain / `expo-secure-store`), **not** plaintext `AsyncStorage`/
   `localStorage`. _(Status: target state; several apps still on AsyncStorage —
   see notes.)_
6. Web session cookies **MUST** be `HttpOnly`, `Secure`, and `SameSite=Lax` (or
   stricter). `Secure` **MUST NOT** be gated on `NODE_ENV` — the site is HTTPS-only.
7. Self-issued tokens **MUST** be HMAC/signature-verified on every request and
   carry an expiry; an unverified or expired token **MUST** be rejected with 401.
8. Self-issued token systems **SHOULD** support revocation (e.g. a per-user
   `tokenVersion`/`validAfter` bumped on logout or password change) rather than
   being purely stateless and unrevocable.

## Rationale

Short access + revocable refresh limits the blast radius of a leak to minutes,
not weeks. Secure on-device storage and hardened cookies stop the easy theft
paths. Revocation is what makes "log out everywhere" and post-compromise response
actually work.

## Acceptance criteria

- [ ] Access token lifetime ≤ 1 hour; refresh issues new access tokens.
- [ ] Refresh token lifetime ≤ ~1 week; an inactive session re-authenticates after that.
- [ ] The client refreshes proactively (before access expiry), not only on a 401.
- [ ] After logout, the prior refresh token can no longer mint access tokens.
- [ ] Session cookies show `HttpOnly; Secure; SameSite=Lax`.
- [ ] A tampered or expired self-issued token returns 401.

## Implementation notes

- **OpenCycle / OpenOutdoor:** Cognito access 1 h; `enableTokenRevocation`; `GlobalSignOut` on web `/logout` and `POST /api/auth/logout`; app `signOut()` revokes; app refreshes ~5 min early (`REFRESH_BEFORE_MS`). **Gaps:** Cognito refresh validity is **90 d** → reduce toward the ~1-week target (and enable rotation); device tokens still in AsyncStorage (move to SecureStore).
- **Emergency:** signed cookie sessions, `Secure: true` always, `HttpOnly`, `SameSite=Lax`.
- **Priorize:** **Gap:** stateless ~90-day tokens, no revocation → shorten to ~1 week, add `tokenVersion`/`validAfter`; move app token to SecureStore (tracked).
- **WebhookNotification:** HMAC access (1 d) + refresh (**1 w**) — meets the target.
