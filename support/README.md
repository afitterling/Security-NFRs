# Support form — all applicable NFRs

Every requirement that bears on the **support / contact / feedback form** —
the public page, the submit endpoint, the double-opt-in confirmation email, and
the "message sent" result page — collected in one place.

Specs here are **copies** of the canonical files in
[`../general-specs/`](../general-specs/); the canonical copy stays authoritative,
this folder is the working set for building the form. One doc was **moved** here
because it is dedicated to this feature (see below). Folder structure mirrors
`general-specs/` so every relative cross-link inside the specs still resolves.

## Moved here (dedicated to the support form)

| Doc | What it is |
|---|---|
| [double-opt-in/feedback-form.md](double-opt-in/feedback-form.md) | The landing feedback/support form itself — form → `POST /support` → confirm email → localized "sent" page, incl. i18n keys. Moved out of `double-opt-in-auth/`. |

## The flow, and what governs each step

| Step | Governing specs |
|---|---|
| Support link is reachable, signed-out, from app + store listing | [UI-006](ui/UI-006-data-privacy-and-support-links.md), [UI-008](design/UI-008-unified-footer.md), [UI-007](design/UI-007-navigable-landing-header.md) |
| The page renders (responsive, labelled, localized, indexable) | [UI-001](design/UI-001-responsive-layout.md), [UI-004](design/UI-004-usability-baseline.md), [A11Y-001](accessibility/A11Y-001-baseline.md), [I18N-001](internationalization/I18N-001-localization.md), [SEO-002](seo/SEO-002-sitemap.md), [SEO-003](seo/SEO-003-metadata-and-social-cards.md) |
| `POST /support` accepts the submission | [SEC-002](security/SEC-002-rate-limiting-and-lockout.md), [SEC-003](security/SEC-003-request-integrity-csrf.md), [SEC-006](security/SEC-006-edge-rate-limiting.md), [DATA-001](data-and-api/DATA-001-api-conventions.md), [gotchas](double-opt-in/gotchas.md) |
| Request is parked + a confirm token is issued | [token-pattern](double-opt-in/token-pattern.md), [DATA-002](data-and-api/DATA-002-storage-conventions.md), [PRIV-001](privacy/PRIV-001-data-minimization-and-retention.md) |
| Confirmation email is sent | [REL-001](reliability/REL-001-observability-and-alerting.md) §5, [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §5, [SEC-004](security/SEC-004-transport-and-at-rest.md) |
| User clicks the link — **GET must be inert** | [link-scanner-safe-confirm](double-opt-in/link-scanner-safe-confirm.md) (**MUST**), [AUTH-002](authentication/AUTH-002-no-tokens-in-urls.md) |
| `POST /support/confirm` relays to support | [link-scanner-safe-confirm](double-opt-in/link-scanner-safe-confirm.md), [SEC-002](security/SEC-002-rate-limiting-and-lockout.md), [REL-002](reliability/REL-002-resilience-and-failure-modes.md) |
| Result page — localized, no enumeration | [SEC-005](security/SEC-005-account-enumeration.md), [I18N-001](internationalization/I18N-001-localization.md), [UI-004](design/UI-004-usability-baseline.md) |
| What the user attaches so support can triage | [REL-005](reliability/REL-005-diagnostics-surface.md) |

## Hard requirements at a glance

1. **GET never mutates.** The emailed confirm link resolves to an inert `GET`
   that renders a `<form method="POST">` + button; only the `POST` relays.
   Corporate link scanners auto-fetch every URL in inbound mail — a relaying GET
   defeats the double opt-in entirely (observed in production, July 2026).
   → [link-scanner-safe-confirm](double-opt-in/link-scanner-safe-confirm.md)
2. **Honeypot + real email validation** on every rendering of the form (hosted
   page *and* landing), rejecting `[\r\n,;<>]` before the value reaches SES.
3. **Rate-limit both endpoints** — submit by IP and IP+email, confirm by IP —
   with a WAF at the CDN in front of the cost-bearing POST.
4. **Reachable without an account, without a paywall, without a consent gate**,
   over HTTPS, with no token/email/session in the URL.
5. **No account enumeration** — the "sent" page must not reveal whether the
   address is registered.
6. **Store only `sha256(token)`**, 24h TTL on the parked row (epoch seconds),
   atomic single-use consume, constant-time compare, one generic failure message.
7. **Localized in all 8 languages** — form, "check your email", and "sent" pages;
   the click can arrive in any locale.
8. **Addresses:** confirmation from `no-reply@sp33c.tech`, forwarded to
   `info@sp33c.tech`, `Reply-To` = the submitter.

## Contents

### double-opt-in/ — the confirmation flow
- [feedback-form.md](double-opt-in/feedback-form.md) — the form itself _(moved)_
- [link-scanner-safe-confirm.md](double-opt-in/link-scanner-safe-confirm.md) — GET must be inert (MUST)
- [token-pattern.md](double-opt-in/token-pattern.md) — generate / store / email / consume
- [gotchas.md](double-opt-in/gotchas.md) — edge cases & security checklist

### ui/ + design/
- [UI-006](ui/UI-006-data-privacy-and-support-links.md) — Data-privacy link & support page _(the core spec)_
- [UI-001](design/UI-001-responsive-layout.md) — Responsive & adaptive layout
- [UI-002](design/UI-002-visual-design-language.md) — Visual design language
- [UI-003](design/UI-003-iconography.md) — Iconography
- [UI-004](design/UI-004-usability-baseline.md) — Usability baseline
- [UI-005](design/UI-005-landing-page-screenshots.md) — Landing page screenshots
- [UI-007](design/UI-007-navigable-landing-header.md) — Navigable landing header (home link from `/support`)
- [UI-008](design/UI-008-unified-footer.md) — Unified footer with legal & support links

### security/
- [SEC-001](security/SEC-001-secrets-management.md) — Secrets management
- [SEC-002](security/SEC-002-rate-limiting-and-lockout.md) — Rate limiting & lockout
- [SEC-003](security/SEC-003-request-integrity-csrf.md) — Request integrity & CSRF
- [SEC-004](security/SEC-004-transport-and-at-rest.md) — Transport & at-rest protection
- [SEC-005](security/SEC-005-account-enumeration.md) — Account-enumeration resistance
- [SEC-006](security/SEC-006-edge-rate-limiting.md) — Edge rate limiting & DDoS _(names contact/support forms explicitly)_

### privacy/
- [PRIV-001](privacy/PRIV-001-data-minimization-and-retention.md) — Data minimization & retention
- [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) — GDPR/DSGVO user rights
- [PRIV-003](privacy/PRIV-003-tracking-consent-att.md) — Tracking consent (ATT)

### reliability/
- [REL-001](reliability/REL-001-observability-and-alerting.md) — Observability & alerting
- [REL-002](reliability/REL-002-resilience-and-failure-modes.md) — Resilience & failure modes
- [REL-004](reliability/REL-004-client-crash-handling.md) — Client crash handling
- [REL-005](reliability/REL-005-diagnostics-surface.md) — Diagnostics surface for support

### data-and-api/
- [DATA-001](data-and-api/DATA-001-api-conventions.md) — API conventions
- [DATA-002](data-and-api/DATA-002-storage-conventions.md) — Storage conventions (TTL, atomic counters)

### authentication/
- [AUTH-001](authentication/AUTH-001-password-credentials.md) — Password credentials
- [AUTH-002](authentication/AUTH-002-no-tokens-in-urls.md) — No tokens in URLs
- [AUTH-003](authentication/AUTH-003-session-token-lifecycle.md) — Session & token lifecycle
- [AUTH-004](authentication/AUTH-004-account-recovery-and-verification.md) — Email verification & account recovery

### accessibility/ · internationalization/ · seo/ · delivery/
- [A11Y-001](accessibility/A11Y-001-baseline.md) — Accessibility baseline (form labels, focus, contrast)
- [I18N-001](internationalization/I18N-001-localization.md) — Localization & locale selection
- [SEO-001](seo/SEO-001-robots-txt.md) — robots.txt
- [SEO-002](seo/SEO-002-sitemap.md) — sitemap.xml
- [SEO-003](seo/SEO-003-metadata-and-social-cards.md) — Page metadata & canonical URLs
- [DEL-002](delivery/DEL-002-environments-and-promotion.md) — Environments & promotion

_Assembled 2026-08-09 from `general-specs/` and `double-opt-in-auth/`._
