# PAGE-003 — The `/support` route

- **Status:** Proposed
- **Group:** Public pages & routes
- **Applies to:** Every app's public web surface. Native clients link *to* this
  route rather than reimplementing the form
  ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §1).
- **Last updated:** 2026-08-26

[UI-006](../ui/UI-006-data-privacy-and-support-links.md) §5 requires a support
page and describes the confirm-by-email behaviour. This spec is the **route
contract** underneath it: the paths, the four states a user passes through, and
what each one must and must not do. The confirmation-token mechanics are the
recipe in [`double-opt-in-auth/`](../../double-opt-in-auth/token-pattern.md);
the assembled build set is [`support/`](../../support/README.md).

## Requirement

### Route & reachability

1. A support/contact page **MUST** be served at a stable path — `/support` — on
   the canonical production host, over **HTTPS**, returning **HTTP 200**, and
   the path **MUST NOT** change between releases: it is filed as the App Store
   support URL and as the Play support contact
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §2–3).
2. The page **MUST** be usable **without an account** — no sign-in, no paywall,
   no tracking-consent gate — and the URL **MUST NOT** carry a token, email, or
   session identifier
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §4,
   [AUTH-002](../authentication/AUTH-002-no-tokens-in-urls.md)).
3. The route **MUST** be linked from the unified footer of every public page
   ([UI-008](../design/UI-008-unified-footer.md) §2) and from the in-app
   settings/about screen
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §1), both deriving
   from one exported `SUPPORT_URL` constant.
4. The same absolute URL **MUST** appear in the store listing's support field
   and in-app — one source of truth, no drift
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §3).

### The four states

5. The flow **MUST** consist of exactly four addressable states, and each
   **MUST** be a real page a user can land on directly:

   | State | Route | What it does |
   |---|---|---|
   | Form | `GET /support` | Collects message + email address |
   | Parked | `POST /support` | Stores the request, emails a confirm link |
   | Confirm | `GET` then `POST /support/confirm` | Inert render, then relay |
   | Sent | result page | Tells the user the message went out |

6. `GET /support/confirm` **MUST** be **inert** — it renders a
   `<form method="POST">` with a confirm button and changes no state. Only the
   `POST` relays the message. This is a **MUST**, not a preference: corporate
   link scanners auto-fetch every URL in inbound mail, and a relaying `GET`
   defeats the double opt-in entirely (observed in production, July 2026)
   → [link-scanner-safe-confirm](../../double-opt-in-auth/link-scanner-safe-confirm.md).
7. On submit the system **MUST** send a confirmation email from
   `no-reply@sp33c.tech` to the address entered, **MUST** forward the confirmed
   message to `info@sp33c.tech` with `Reply-To` set to the submitter, and
   **MUST** then show a page stating the email was sent
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §5).
8. No state in the flow **MUST** reveal whether the submitted address belongs to
   an existing account — same copy, same status, same timing for a known and an
   unknown address ([SEC-005](../security/SEC-005-account-enumeration.md)).

### The public write endpoint

9. `POST /support` is an unauthenticated, cost-bearing, mail-sending endpoint
   and **MUST** be treated as one:
   - rate-limited **by IP and by IP+email**, with `/support/confirm` limited by
     IP ([SEC-002](../security/SEC-002-rate-limiting-and-lockout.md));
   - fronted by an edge/WAF rule before the cost is incurred
     ([SEC-006](../security/SEC-006-edge-rate-limiting.md));
   - protected against cross-site submission
     ([SEC-003](../security/SEC-003-request-integrity-csrf.md));
   - conforming to the house API shape
     ([DATA-001](../data-and-api/DATA-001-api-conventions.md)).
10. Every rendering of the form (hosted page *and* landing embed) **MUST** carry
    a **honeypot field** and **MUST** validate the email server-side, rejecting
    `[\r\n,;<>]` before the value reaches the mailer — header injection into SES
    is the failure this prevents.
11. The parked request **MUST** store only `sha256(token)`, carry an absolute
    TTL (**24h**, epoch seconds, as the table TTL so unconfirmed rows
    self-sweep), be consumed **atomically and once**, and be compared in
    **constant time**, with a single generic failure message for expired,
    unknown, and already-used tokens
    ([token-pattern](../../double-opt-in-auth/token-pattern.md),
    [DATA-002](../data-and-api/DATA-002-storage-conventions.md)).
12. The message body and address are personal data: they **MUST** be retained
    only as long as the support interaction needs them and **MUST** be covered
    by the retention statement on `/privacy`
    ([PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md),
    [PAGE-001](PAGE-001-privacy-route.md) §6–7).

### Rendering

13. All four states **MUST** be localized in every baseline locale and
    auto-detect the visitor's language — the confirm click can arrive in any
    locale, from a mail client, hours later
    ([I18N-001](../internationalization/I18N-001-localization.md)).
14. The form **MUST** have real labels, visible focus, and errors tied to their
    fields; it **MUST** stay usable at every breakpoint and at increased font
    scale ([A11Y-001](../accessibility/A11Y-001-baseline.md),
    [UI-001](../design/UI-001-responsive-layout.md),
    [UI-004](../design/UI-004-usability-baseline.md)).
15. The form page **SHOULD** tell the user what to attach so support can triage
    — app version, build, platform, and the diagnostics identifier
    ([REL-005](../reliability/REL-005-diagnostics-surface.md)).
16. The page **MUST** carry a link to `/privacy`
    ([PAGE-001](PAGE-001-privacy-route.md)) next to the submit control, because
    submitting sends personal data.

### Indexing & availability

17. `GET /support` **SHOULD** be indexable with a unique localized title and
    canonical URL; `/support/confirm` and the sent page **MUST** be `noindex`
    and **MUST NOT** appear in the sitemap
    ([SEO-002](../seo/SEO-002-sitemap.md),
    [SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §8).
18. The form page **MUST** still render when the mailer or datastore is
    unavailable, and a failed submit **MUST** produce an honest error rather
    than a false "sent" page
    ([REL-002](../reliability/REL-002-resilience-and-failure-modes.md)).
19. Confirm-email delivery failures **MUST** be observable — a submit that never
    results in a sent mail is invisible to the user and to support unless it
    alerts ([REL-001](../reliability/REL-001-observability-and-alerting.md) §5).
20. Non-production copies **MUST NOT** be indexable and **MUST NOT** be the URL
    filed with the stores ([SEO-001](../seo/SEO-001-robots-txt.md) §5,
    [DEL-002](../delivery/DEL-002-environments-and-promotion.md)).

## Rationale

The support route is the only public, unauthenticated, mail-sending write
endpoint most of these apps expose. That makes it simultaneously a store-review
requirement (a missing or dead support URL blocks a release), a compliance
surface (it is where erasure and export requests actually arrive —
[PAGE-001](PAGE-001-privacy-route.md) §11), and the most attractive abuse target
in the app: free outbound mail, paid for by us.

Its two real failure modes are both quiet. A relaying `GET /support/confirm`
turns the double opt-in into a no-op the moment a corporate scanner touches the
mail — it looks like it works, and it confirms every message automatically. An
unthrottled `POST /support` looks fine until someone finds it. Both are cheap to
get right at build time and awkward to retrofit.

## Acceptance criteria

- [ ] `GET https://<prod-host>/support` → 200, HTTPS, anonymous, no consent gate, no identifier in URL.
- [ ] Footer link, in-app support link, and the store-listing support URL are byte-identical.
- [ ] `GET /support/confirm?...` mutates nothing — verified by fetching it twice and confirming no mail is relayed.
- [ ] Only `POST /support/confirm` relays; the token is single-use and dies on second use.
- [ ] Confirmation mail comes from `no-reply@sp33c.tech`; the relay reaches `info@sp33c.tech` with `Reply-To` = submitter.
- [ ] The sent page is identical for a registered and an unregistered address.
- [ ] `POST /support` is rate-limited by IP and IP+email, with an edge rule in front; `/support/confirm` limited by IP.
- [ ] Honeypot present on every rendering; an email containing `\r`, `\n`, `<`, or `;` is rejected server-side.
- [ ] Only `sha256(token)` is stored; the row carries a 24h epoch-second TTL and is consumed atomically.
- [ ] Expired, unknown, and reused tokens produce one identical message.
- [ ] All four states render in every baseline locale, including a confirm click in a locale different from the submit.
- [ ] Form labelled and operable at 320px and 200% font scale; `/privacy` linked at the submit control.
- [ ] `/support` indexable; `/support/confirm` and the sent page `noindex` and absent from the sitemap.
- [ ] A mailer outage yields an error state, never a false "sent"; the failure alerts.

## Implementation notes

- The build set for this route is assembled in
  [`support/`](../../support/README.md) — every applicable spec copied into one
  folder, plus the form and token docs. Build from there; fix bugs here.
- **Reference implementation:** `POST /support` parks a `SupportRequest`,
  `sendSupportConfirmEmail` mails `…/support/confirm?id=&token=`, `GET` renders
  the inert button, `POST` does the atomic fetch-and-delete then forwards —
  `sst/src/index.ts`, `sst/src/db.ts`, `sst/src/email.ts` in `threethings`.
- Keep the local-dev fallback that logs the confirm link to the console instead
  of sending mail; it is what makes the flow testable without a verified SES
  identity.
- **Per-app status:** implemented in **Priorize** (`/support` + confirm, in
  production). Other apps in scope **have not been audited** against this route
  — treat as a gap until checked.
- Related: [[PAGE-001-privacy-route]], [[PAGE-004-password-reset-routes]],
  [[UI-006-data-privacy-and-support-links]], [[UI-008-unified-footer]].
