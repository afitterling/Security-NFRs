# Privacy page (`/privacy`) — all applicable NFRs

Every requirement that bears on the **privacy notice route** — the public
`/privacy` page, the links that point at it (footer, in-app settings, store
listing), the imprint beside it, and the user-rights actions it has to make
reachable — collected in one place.

Specs here are **copies** of the canonical files in
[`../general-specs/`](../general-specs/); the canonical copy stays authoritative,
this folder is the working set for building the page. Folder structure mirrors
`general-specs/` so every relative cross-link inside the specs still resolves.
Same convention as [`../support/`](../support/).

## The route specs

| Doc | What it is |
|---|---|
| [pages/PAGE-001-privacy-route.md](pages/PAGE-001-privacy-route.md) | **The route spec.** The one doc here that is *about* `/privacy` itself: stable path, reachability, required content, rights actions, rendering, indexing posture, availability. |
| [pages/PAGE-002-imprint-route.md](pages/PAGE-002-imprint-route.md) | **The imprint spec.** What the Impressum beside this page has to contain, and how it has to be labelled and linked. |
| [pages/PAGE-003](pages/PAGE-003-support-route.md) · [pages/PAGE-004](pages/PAGE-004-password-reset-routes.md) | Included so cross-links resolve — the support route (where rights requests land) and the reset routes. |

Both were drafted here and are now **canonical** in
[`../general-specs/pages/`](../general-specs/pages/); these are copies. `PAGE-001`
was `PRIV-004` while it was bundle-local — that ID is retired and never reused.

## Copied in (dedicated background, not a numbered spec)

| Doc | What it is |
|---|---|
| [store-label/non-tracking-purchases.md](store-label/non-tracking-purchases.md) | ATT vs. App Store privacy label under Guideline 5.1.2(i) — what the privacy page's tracking section has to agree with, and the repeat-rejection trap (stale `NSUserTrackingUsageDescription` in the shipped binary). Copied from `../IAP-Subscriptions/non-tracking-purchases/`; **not moved**, because `IAP-Subscriptions/app-store-iap-setup/`, `IAP-Subscriptions/revenuecat-integration/` and `IAP-Subscriptions/storekit-paywall-gating/` all reference it in place. |

## The page, and what governs each part

| Part | Governing specs |
|---|---|
| The route exists at a stable path, 200 over HTTPS, signed-out | [PAGE-001](pages/PAGE-001-privacy-route.md) §1–3, [UI-006](ui/UI-006-data-privacy-and-support-links.md) §2/§4, [AUTH-002](authentication/AUTH-002-no-tokens-in-urls.md) |
| It's discoverable — footer, header, in-app, store listing | [UI-008](design/UI-008-unified-footer.md), [UI-007](design/UI-007-navigable-landing-header.md), [UI-006](ui/UI-006-data-privacy-and-support-links.md) §1/§3 |
| Imprint (Impressum) sits beside it and is linked | [PAGE-002](pages/PAGE-002-imprint-route.md), [PAGE-001](pages/PAGE-001-privacy-route.md) §4, [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §1, [UI-008](design/UI-008-unified-footer.md) §2 |
| What the copy must say — collection, purpose, retention, third parties | [PRIV-001](privacy/PRIV-001-data-minimization-and-retention.md), [PAGE-001](pages/PAGE-001-privacy-route.md) §6–10 |
| Lawful basis, revocable consent, verified transactional sender | [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §4–5 |
| Tracking section ↔ store label ↔ shipped binary agree | [PRIV-003](privacy/PRIV-003-tracking-consent-att.md) §5, [store-label](store-label/non-tracking-purchases.md), [UI-006](ui/UI-006-data-privacy-and-support-links.md) §6 |
| Rights are actionable — erasure, export, withdraw consent | [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §2–4, [UI-006](ui/UI-006-data-privacy-and-support-links.md) §5 |
| Any request form on the page (deletion/export by email) | [SEC-002](security/SEC-002-rate-limiting-and-lockout.md), [SEC-003](security/SEC-003-request-integrity-csrf.md), [SEC-005](security/SEC-005-account-enumeration.md), [SEC-006](security/SEC-006-edge-rate-limiting.md) |
| It renders — responsive, readable long-form, accessible | [UI-001](design/UI-001-responsive-layout.md), [UI-002](design/UI-002-visual-design-language.md), [UI-004](design/UI-004-usability-baseline.md), [A11Y-001](accessibility/A11Y-001-baseline.md) |
| It's localized and auto-detects the visitor's language | [I18N-001](internationalization/I18N-001-localization.md) |
| Indexing posture is consistent everywhere | [SEO-001](seo/SEO-001-robots-txt.md), [SEO-002](seo/SEO-002-sitemap.md), [SEO-003](seo/SEO-003-metadata-and-social-cards.md), [SEO-004](seo/SEO-004-structured-data.md) |
| Transport & serving | [SEC-004](security/SEC-004-transport-and-at-rest.md), [SEC-006](security/SEC-006-edge-rate-limiting.md) |
| It stays up, and preview copies stay out of the index | [REL-001](reliability/REL-001-observability-and-alerting.md), [REL-002](reliability/REL-002-resilience-and-failure-modes.md), [DEL-002](delivery/DEL-002-environments-and-promotion.md) |

## Hard requirements at a glance

1. **One URL, three places.** The footer link, the in-app settings link, and the
   store-listing privacy-policy field must be the *same* absolute HTTPS URL,
   sourced from one constant. Drift here is the classic App Review rejection.
   → [PAGE-001](pages/PAGE-001-privacy-route.md) §5, [UI-006](ui/UI-006-data-privacy-and-support-links.md) §3
2. **Reachable before any gate.** No account, no paywall, **no consent gate** —
   the page has to render for someone who has decided nothing yet, and with
   consent-gated third-party scripts blocked.
3. **The copy must match the system.** Every retention period stated must
   correspond to a TTL that is actually set; every processor receiving data must
   be named; nothing may be claimed that the app doesn't do.
   → [PRIV-001](privacy/PRIV-001-data-minimization-and-retention.md) §2/§5
4. **Tracking posture is a three-way agreement** — page copy, store privacy
   label, shipped binary. For a no-tracking app: no `collectDeviceIdentifiers()`,
   no ATT plugin, and `NSUserTrackingUsageDescription` absent from the *shipped*
   Info.plist (introspect merges the committed `ios/` project — verify both
   layers). → [store-label](store-label/non-tracking-purchases.md)
5. **Erasure has to be real and findable.** Delete-or-irreversibly-anonymize, not
   "disable login", and the page must name a working way to trigger it.
   → [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §2
6. **Imprint alongside.** Privacy notice *and* imprint/Impressum, both linked
   from the unified footer on every page.
7. **Localized in all nine baseline locales** (en, de, fr, es, it, zh, ja, ms,
   id), auto-detected from `Accept-Language`, `?lang=` overriding and persisting
   — with the caveat that unreviewed machine-translated legal copy should fall
   back to a reviewed language rather than ship.
8. **Pick an indexing posture and hold it.** Indexable (canonical + sitemap +
   metadata) or `noindex` (absent from sitemap, no canonical/social tags) — never
   half of each.
9. **No identifiers in the URL.** `?lang=` is the only query parameter this route
   takes.
10. **Static render, monitored.** No datastore or third-party call needed to
    serve it; an uptime check alerts on non-200.

## Contents

### pages/ — the routes themselves
- [PAGE-001](pages/PAGE-001-privacy-route.md) — The `/privacy` route _(the core spec)_
- [PAGE-002](pages/PAGE-002-imprint-route.md) — The `/imprint` route (Impressum)
- [PAGE-003](pages/PAGE-003-support-route.md) — The `/support` route _(where rights requests land)_
- [PAGE-004](pages/PAGE-004-password-reset-routes.md) — The `/forgot` + `/reset` routes _(cross-link target)_

### privacy/
- [PRIV-001](privacy/PRIV-001-data-minimization-and-retention.md) — Data minimization & retention
- [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) — GDPR/DSGVO user rights _(the legal core)_
- [PRIV-003](privacy/PRIV-003-tracking-consent-att.md) — Tracking consent (ATT) & label truthfulness

### store-label/
- [non-tracking-purchases.md](store-label/non-tracking-purchases.md) — ATT vs. privacy label, Guideline 5.1.2(i)

### ui/ + design/
- [UI-006](ui/UI-006-data-privacy-and-support-links.md) — Data-privacy link & support page _(the core spec)_
- [UI-001](design/UI-001-responsive-layout.md) — Responsive & adaptive layout
- [UI-002](design/UI-002-visual-design-language.md) — Visual design language
- [UI-003](design/UI-003-iconography.md) — Iconography
- [UI-004](design/UI-004-usability-baseline.md) — Usability baseline
- [UI-005](design/UI-005-landing-page-screenshots.md) — Landing page screenshots _(link target of UI-002/UI-007)_
- [UI-007](design/UI-007-navigable-landing-header.md) — Navigable landing header (home link from `/privacy`)
- [UI-008](design/UI-008-unified-footer.md) — Unified footer with legal & support links

### security/
- [SEC-001](security/SEC-001-secrets-management.md) — Secrets management
- [SEC-002](security/SEC-002-rate-limiting-and-lockout.md) — Rate limiting & lockout
- [SEC-003](security/SEC-003-request-integrity-csrf.md) — Request integrity & CSRF
- [SEC-004](security/SEC-004-transport-and-at-rest.md) — Transport & at-rest protection
- [SEC-005](security/SEC-005-account-enumeration.md) — Account-enumeration resistance
- [SEC-006](security/SEC-006-edge-rate-limiting.md) — Edge rate limiting & DDoS

### authentication/
- [AUTH-002](authentication/AUTH-002-no-tokens-in-urls.md) — No tokens in URLs _(applies directly)_
- [AUTH-001](authentication/AUTH-001-password-credentials.md) · [AUTH-003](authentication/AUTH-003-session-token-lifecycle.md) · [AUTH-004](authentication/AUTH-004-account-recovery-and-verification.md) — included so cross-links resolve

### reliability/ · delivery/
- [REL-001](reliability/REL-001-observability-and-alerting.md) — Observability & alerting (uptime of the privacy URL)
- [REL-002](reliability/REL-002-resilience-and-failure-modes.md) — Resilience & failure modes
- [REL-005](reliability/REL-005-diagnostics-surface.md) — Diagnostics surface for support _(cross-link target)_
- [DATA-001](data-and-api/DATA-001-api-conventions.md) · [DATA-002](data-and-api/DATA-002-storage-conventions.md) — API & storage conventions _(cross-link targets)_
- [DEL-002](delivery/DEL-002-environments-and-promotion.md) — Environments & promotion

### accessibility/ · internationalization/ · seo/
- [A11Y-001](accessibility/A11Y-001-baseline.md) — Accessibility baseline (heading order, contrast, font scaling)
- [I18N-001](internationalization/I18N-001-localization.md) — Localization & locale selection
- [SEO-001](seo/SEO-001-robots-txt.md) — robots.txt
- [SEO-002](seo/SEO-002-sitemap.md) — sitemap.xml
- [SEO-003](seo/SEO-003-metadata-and-social-cards.md) — Page metadata & canonical URLs
- [SEO-004](seo/SEO-004-structured-data.md) — Structured data (JSON-LD)

_Assembled 2026-08-09 from `general-specs/` and `IAP-Subscriptions/non-tracking-purchases/`._
