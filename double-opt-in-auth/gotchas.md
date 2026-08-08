# Gotchas & security checklist

Hard-won rules for the confirmation-link flows. Most are already honored by the
`/support` and `/forgot` reference implementations — keep them when adding
signup confirmation and the feedback form.

## Link scanners — GET must be inert (learned the hard way)
- **Never mutate state or send mail on the `GET` a link resolves to.** Corporate
  inbound-mail security (Defender SafeLinks, Proofpoint, Mimecast) auto-fetches
  every URL in inbound mail; a `GET` that relayed/verified got auto-triggered by
  those scanners, defeating the whole double opt-in. Render a button; do the work
  in a `POST`. Full NFR: [link-scanner-safe-confirm.md](link-scanner-safe-confirm.md).
- **Honeypot every form** that feeds an email path (a hidden `website` field);
  drop silently when filled. Add it to *all* renderings — hosted page and landing.
- **Validate the email for real** (`emailOk` regex, len ≤ 254) and reject
  `[\r\n,;<>]` before it becomes an SES `Reply-To`/`To` — not just `includes("@")`.

## Tokens & storage
- **Never persist the raw token** — store only `sha256(token)`. The raw value
  exists solely in the emailed URL. A leaked DB row must be useless.
- **Never log the raw token in production.** The local-dev console fallback that
  prints the link is gated on *not* running on Lambda — keep that gate.
- Token from a CSPRNG (`crypto.randomBytes`), ≥32 bytes, base64url.

## Verification
- **Constant-time compare** (`crypto.timingSafeEqual`) on the hashes — never
  `===` on the secret. Guard the length first (`timingSafeEqual` throws on
  unequal-length buffers).
- **One generic failure message** ("invalid or has expired"). Don't distinguish
  expired vs. wrong vs. already-used — that's an oracle.

## Single-use & expiry
- **Atomic consume** for parked rows: `DeleteCommand … ReturnValues: ALL_OLD`,
  so a double-click / race can't process twice. For record-attached tokens
  (reset, signup), clear the hash in the same write that applies the effect.
- **TTL units bite:** parked-row `expiresAt` is **epoch seconds** (DynamoDB TTL);
  record fields (`resetExpiresAt`, `confirmExpiresAt`) are **epoch ms**. Convert
  when comparing (`rec.expiresAt * 1000 >= Date.now()`).
- DynamoDB TTL deletion is **lazy** (can lag hours) — always re-check expiry in
  code; don't rely on the sweep for security.

## Anti-enumeration
- Endpoints that take an email (`/forgot`, signup) must return an **identical
  neutral response** whether or not the address exists, and run `decoyVerify` on
  the not-found path to flatten timing.
- Signup "already exists" must not become an oracle — for an *unconfirmed*
  existing account, resend the link rather than hard-failing differently.

## Rate limiting
- Throttle **both** the request endpoint (by IP and IP+email) **and** the
  confirm/click endpoint (by IP). The reference impl uses
  `allow(scope, { limit, windowSec }, now)`; give each flow its own scope prefix
  (`support:`, `forgot:`, `confirm:`, `supportconfirm:`).

## Email delivery
- SES must be **out of the sandbox** (and the domain/identity verified in the
  function's region) to mail arbitrary recipients in prod. Until then only
  verified addresses receive mail; dev logs the link.
- Send from a verified identity (`RESET_FROM`); set **Reply-To** to the user for
  forwarded messages so support can reply directly.

## Landing / localization
- Confirmation **landing pages must be localized** in all 8 languages
  (en/de/zh/ms/id/fr/es/pl) — the click can arrive in any locale. Detect via the
  existing `detectLang`/`?lang=` mechanism.
- If `/support/confirm` redirects to the Remix landing, the Auth function needs
  `LANDING_BASE_URL` in its env (currently only an iosapp var).
- Don't put the user's email or token in a redirect URL that ends up in browser
  history / referrer headers.

## Signup-specific
- Changing signup to *not* issue a session is a **breaking client change** — the
  landing form action and the native PKCE path must both handle a `pending`
  response (show "check your email") instead of expecting a token.
- Grandfather pre-feature accounts (no `confirmedAt`) so existing users aren't
  locked out at the login guard.
