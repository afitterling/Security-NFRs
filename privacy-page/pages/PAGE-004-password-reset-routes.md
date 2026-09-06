# PAGE-004 — The `/forgot` + `/reset` routes

- **Status:** Proposed
- **Group:** Public pages & routes
- **Applies to:** Every app with email-and-password accounts. Apps whose only
  sign-in is federated or device-key based (**Emergency**) are out of scope.
- **Last updated:** 2026-08-26

[AUTH-004](../authentication/AUTH-004-account-recovery-and-verification.md) §2
requires that a self-service forgot-password flow **exist**. This spec is the
route contract for it: the two paths, the four states, and the handful of things
that are easy to get subtly wrong — an entry link nobody added, a `GET` that
burns the token before the user sees the form, a reset URL that leaks through
the `Referer` header.

## Requirement

### Entry point

1. **Every** rendering of a login form **MUST** carry a visible, localized
   **"Forgot password?"** link to `/forgot`, in login mode, near the password
   field. A reset flow that exists on the backend but is linked from nowhere
   does not satisfy
   [AUTH-004](../authentication/AUTH-004-account-recovery-and-verification.md) §2 —
   this includes inline landing-page login forms, not just a dedicated
   `/login` route.
2. The link **MUST NOT** prefill or pass the typed email in the URL. `/forgot`
   collects the address itself; putting it in the query string leaks it into
   browser history, server logs, and the `Referer` header.
3. `/forgot` **MUST** also be reachable directly, signed-out, at a stable path
   over HTTPS.

### The four states

4. The flow **MUST** consist of exactly four addressable states:

   | State | Route | What it does |
   |---|---|---|
   | Request | `GET /forgot` | Collects the email address |
   | Requested | `POST /forgot` | Issues a token, mails the link, shows a neutral notice |
   | Set | `GET` then `POST /reset` | Inert render of the new-password form, then the change |
   | Done | result page | Confirms the change; points at sign-in |

5. `GET /reset?token=…` **MUST** be **inert**: it validates that a token is
   *present* and renders the new-password form, and **MUST NOT** consume,
   invalidate, or rotate the token. Corporate link scanners auto-fetch every URL
   in inbound mail; a `GET` that burns the token means the user's reset link is
   already dead when they click it
   ([link-scanner-safe-confirm](../../double-opt-in-auth/link-scanner-safe-confirm.md)).
   Only `POST /reset` consumes it.
6. Both `GET /forgot` and `GET /reset` **MUST** render for an anonymous visitor
   with no consent gate, and **MUST** work when the user's session is expired or
   absent — which is the normal case.

### Neutrality

7. `POST /forgot` **MUST** return the **same page, message, and status** whether
   or not the address is registered — *"If that email has an account, a code is
   on its way."* — never *"no such user"*
   ([SEC-005](../security/SEC-005-account-enumeration.md),
   [AUTH-004](../authentication/AUTH-004-account-recovery-and-verification.md) §3).
8. The **timing** of the two paths **MUST** be equalized. The unknown-address
   path **MUST** perform decoy work (a `decoyVerify` against a dummy hash)
   rather than returning early — an unauthenticated attacker measuring response
   time otherwise gets the enumeration oracle §7 just closed.
9. Expired, unknown, already-used, and malformed tokens at `/reset` **MUST**
   produce **one** generic failure message with a link back to `/forgot`. The
   page **MUST NOT** distinguish "expired" from "never existed".

### The token

10. The reset token **MUST** be generated from a CSPRNG (32 bytes, base64url),
    and the system **MUST** store only `sha256(token)` plus an absolute expiry —
    never the raw value
    ([token-pattern](../../double-opt-in-auth/token-pattern.md)).
11. The token TTL **MUST** be short — **1 hour** — and comparison **MUST** be
    constant-time (`timingSafeEqual`).
12. The token **MUST** be single-use: the stored hash **MUST** be cleared
    atomically as part of the password change, so a replayed link fails even if
    the mail is later forwarded or a scanner re-fetches it.
13. Issuing a new token **MUST** invalidate any outstanding one for that
    account — a user who clicks "resend" twice **MUST NOT** be left with two
    live reset paths.
14. The raw token appearing in the `/reset` URL is **not** a violation of
    [AUTH-002](../authentication/AUTH-002-no-tokens-in-urls.md) §1, which governs
    access/ID/refresh tokens. It is a single-use, short-lived recovery
    credential. Its URL-borne consequences **MUST** still be contained:
    - the `/reset` response **MUST** send `Referrer-Policy: no-referrer`, so the
      token is not handed to any third-party asset the page loads;
    - `/reset` **MUST NOT** load third-party scripts, fonts, or images;
    - the token **MUST NOT** be written to application logs or analytics;
    - after a successful `POST`, the flow **MUST** redirect to a
      **token-free** URL, so the address bar and history do not retain it.

### The change

15. The new password **MUST** be validated against
    [AUTH-001](../authentication/AUTH-001-password-credentials.md) at `POST /reset`
    — server-side, with the same rules as sign-up. Client-side checking is a
    convenience, never the enforcement point.
16. A successful reset **MUST** revoke existing sessions and refresh tokens for
    the account ([AUTH-003](../authentication/AUTH-003-session-token-lifecycle.md),
    [AUTH-004](../authentication/AUTH-004-account-recovery-and-verification.md) §5),
    and the done page **MUST** say so — the user resetting a password because it
    may be compromised needs to know the other sessions are gone.
17. A successful reset **SHOULD** send a **notification email** to the account
    address stating that the password changed and what to do if it wasn't them.
    That mail **MUST NOT** contain the new password or the token.
18. A successful reset **SHOULD NOT** silently sign the user in; it **SHOULD**
    hand them to the sign-in form. Where the app does sign them in, it **MUST**
    mint a fresh session after the revocation in §16, never reuse the old one.
19. Where the account is unverified, a completed reset **MAY** be treated as
    proof of address ownership and mark it verified — but **MUST NOT** grant any
    entitlement beyond that.

### Rate limiting

20. `POST /forgot` **MUST** be rate-limited **by IP and by IP+email**
    ([SEC-002](../security/SEC-002-rate-limiting-and-lockout.md)) — it is an
    unauthenticated endpoint that sends mail on demand, so it is both an
    inbox-flooding weapon aimed at a user and a cost-bearing endpoint aimed at
    us. An edge rule **MUST** sit in front of it
    ([SEC-006](../security/SEC-006-edge-rate-limiting.md)).
21. `POST /reset` **MUST** be rate-limited by IP **and** by token, so a valid
    token cannot be brute-forced and a stolen one cannot be used to grind
    passwords past [AUTH-001](../authentication/AUTH-001-password-credentials.md).
22. Throttling **MUST NOT** be reported differently for known and unknown
    addresses — the throttle response is part of the neutrality surface in §7.

### Rendering, indexing, availability

23. All four states **MUST** be localized in every baseline locale and
    auto-detect the visitor's language — the reset click arrives from a mail
    client, possibly hours later, in any locale
    ([I18N-001](../internationalization/I18N-001-localization.md)).
24. Both forms **MUST** have real labels, visible focus, and errors tied to
    their fields; the password field **MUST** be `type="password"` with
    `autocomplete="new-password"` at `/reset` and **SHOULD** offer a
    show-password toggle
    ([A11Y-001](../accessibility/A11Y-001-baseline.md),
    [UI-004](../design/UI-004-usability-baseline.md)).
25. All four states **MUST** be `noindex` and **MUST NOT** appear in
    `sitemap.xml` ([SEO-002](../seo/SEO-002-sitemap.md),
    [SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §8).
26. Reset-mail delivery failures **MUST** be observable
    ([REL-001](../reliability/REL-001-observability-and-alerting.md) §5): a user
    locked out by a silently failing mailer will not report it as a mail
    problem, and the flow gives them no signal — §7 requires the same neutral
    notice whether or not anything was sent.
27. A mailer or datastore outage **MUST** produce an honest error, never a
    neutral notice that implies a mail went out when none did. Neutrality about
    *whether the address exists* **MUST NOT** be used to hide *that the system
    failed*.

## Rationale

Password reset is the flow where security requirements and usability
requirements pull hardest against each other, and where the compromises are
invisible until exploited. It is also, structurally, **an authentication bypass
with a legitimate purpose**: anyone who reaches the end of it holds the account.

The three failures this route keeps producing are all quiet. A reset flow ships
with no link to it, so users mail support instead — the backend works, the
feature does not exist. A `GET /reset` consumes the token, so every user behind a
scanning mail gateway gets a dead link and no explanation. And an unthrottled,
non-neutral `POST /forgot` turns into an account-enumeration oracle and a mail
cannon at the same time.

§14 is the one that reads like an exception and isn't. A recovery link must carry
its credential in a URL — that is what makes it clickable from an email — so the
containment has to be explicit: no referrer, no third-party assets, no logging,
and a token-free URL the moment the flow completes.

## Acceptance criteria

- [ ] Every login form — including inline landing-page forms — shows a localized "Forgot password?" link to `/forgot`.
- [ ] No email address appears in the `/forgot` URL from any entry point.
- [ ] `POST /forgot` for an unknown address returns the same page, status, and message as for a known one.
- [ ] Response times for known and unknown addresses are indistinguishable (decoy work runs on the unknown path).
- [ ] `GET /reset?token=…` fetched twice still leaves the token usable; only `POST` consumes it.
- [ ] Expired, unknown, reused, and malformed tokens all return one identical message.
- [ ] Only `sha256(token)` is stored; TTL is 1h; comparison is constant-time.
- [ ] Requesting a second reset invalidates the first token.
- [ ] `/reset` sends `Referrer-Policy: no-referrer` and loads no third-party asset; the token appears in no log line.
- [ ] After a successful reset the browser lands on a URL containing no token.
- [ ] The new password is validated server-side against AUTH-001.
- [ ] Existing sessions and refresh tokens are revoked; the done page says so; a notification email is sent containing neither password nor token.
- [ ] `POST /forgot` is limited by IP and IP+email with an edge rule in front; `POST /reset` is limited by IP and by token.
- [ ] Throttled responses are identical for known and unknown addresses.
- [ ] All four states render in every baseline locale, including a reset click in a locale different from the request.
- [ ] All four states are `noindex` and absent from the sitemap.
- [ ] A mailer outage surfaces an error and alerts — not a neutral "check your email".

## Implementation notes

- The token mechanics are shared with the support flow — same generate/store/
  email/consume recipe, different TTL (**1h** here, **24h** there):
  [`double-opt-in-auth/token-pattern.md`](../../double-opt-in-auth/token-pattern.md).
  The entry-link work is written up in
  [`forgot-password-link.md`](../../double-opt-in-auth/forgot-password-link.md),
  including the `forgotPassword` i18n key in all 8 languages.
- **Reference implementation:** `GET/POST /forgot` + `GET/POST /reset` in
  `sst/src/index.ts`; `sendResetEmail` in `email.ts`; `resetHash` /
  `resetExpiresAt` stored on the user row and cleared on use (`threethings`).
- Keep the local-dev fallback that logs the reset link to the console instead of
  sending mail — without it the flow is untestable locally.
- **Per-app status:**
  - **Priorize:** backend implemented and in production. **Gap:** §1 — the
    landing login form still has no "Forgot password?" link, so the flow is
    unreachable for most users.
  - **OpenCycle / OpenOutdoor:** Cognito-hosted `/forgot` + `/reset` with
    `preventUserExistenceErrors=ENABLED`. Audit against §5 (inert `GET`) and
    §14 (referrer containment), which Cognito's hosted pages do not guarantee.
  - **WebhookNotification:** implemented in `lib/auth.ts` + API routes; not yet
    audited against this spec.
  - **Emergency:** N/A — no email/password sign-in.
- Related: [[PAGE-003-support-route]],
  [[AUTH-004-account-recovery-and-verification]],
  [[AUTH-001-password-credentials]], [[AUTH-002-no-tokens-in-urls]],
  [[SEC-005-account-enumeration]].
