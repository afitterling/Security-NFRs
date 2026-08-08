# SEC-003 — Request integrity & CSRF

- **Status:** Adopted
- **Group:** Security
- **Applies to:** Every app that mutates state from a browser (form POSTs or
  cookie-authenticated JSON APIs).
- **Last updated:** 2026-06-16

## Requirement

1. Server-rendered state-changing forms **MUST** carry CSRF protection — a
   double-submit token (HttpOnly, `SameSite` cookie + matching hidden field,
   compared in constant time) or an equivalent.
2. Cookie-authenticated JSON APIs **MUST** reject state-changing requests that
   aren't `application/json` (a cross-site form cannot set that content type
   without a CORS preflight), in addition to `SameSite` cookies.
3. All untrusted input **MUST** be validated and bounded: length caps on every
   field, type/shape checks on JSON bodies, and rejection (not truncation that
   changes meaning) of malformed input.
4. All user-controlled output rendered into HTML **MUST** be escaped to prevent
   XSS.
5. Redirect/callback targets **MUST** be validated against an allowlist
   (see [AUTH-002](../authentication/AUTH-002-no-tokens-in-urls.md)); open
   redirects are prohibited.

## Rationale

CSRF lets a third-party site act as the logged-in user; missing input bounds and
escaping invite injection and resource abuse. These are cheap, universal guards.

## Acceptance criteria

- [ ] A form POST without a valid `_csrf` token is rejected.
- [ ] A cookie-auth mutation with `Content-Type: text/plain` is rejected (415).
- [ ] Oversized field values are rejected/capped; malformed JSON returns 400.
- [ ] Reflected user input appears HTML-escaped in responses.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `csrf.ts` double-submit on login/signup/verify/forgot/reset; `esc()` on rendered values; field length caps; pinned redirect allowlist.
- **OpenCycle (JSON API):** `requireJson()` content-type guard on mutations.
- **Emergency:** Remix actions + `SameSite=Lax` session; input length caps on the login form.
