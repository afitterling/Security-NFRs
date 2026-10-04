# Landing page (`LAND`)

The public marketing site of each app: the page at `/`, the navigation it shares
with its sub-pages, and the plumbing every public host needs. These specs were
extracted from the **WebhookPush** landing site (`webhook/sst/landing`), which
is the reference implementation, and then generalized. Each spec's
implementation notes show where WebhookPush meets it and where it doesn't yet.

| ID | Title | Status |
|----|-------|--------|
| [LAND-001](LAND-001-landing-page-anatomy.md) | Landing page anatomy & section anchors | Proposed |
| [LAND-002](LAND-002-calls-to-action-and-store-links.md) | Calls to action & store links | Proposed |
| [LAND-003](LAND-003-responsive-navigation.md) | Responsive navigation & off-canvas drawer | Proposed |
| [LAND-004](LAND-004-language-and-locale-persistence.md) | Language switcher, locale persistence & cache safety | Proposed |
| [LAND-005](LAND-005-pricing-section.md) | Pricing section | Proposed |
| [LAND-006](LAND-006-showcase-media-pipeline.md) | Showcase media pipeline & responsive images | Proposed |
| [LAND-007](LAND-007-security-overview-page.md) | Public security overview (`/security`) | Proposed |
| [LAND-008](LAND-008-docs-and-setup-guides.md) | Documentation & setup-guide pages | Proposed |
| [LAND-009](LAND-009-document-shell.md) | Document shell: head, focus, motion & zoom | Proposed |
| [LAND-010](LAND-010-icons-manifest-and-well-known-files.md) | Site icons, manifest & well-known files | Proposed |
| [LAND-011](LAND-011-not-found-scanners-and-legacy-redirects.md) | Not-found pages, scanner traffic & legacy redirects | Proposed |
| [LAND-012](LAND-012-truthful-marketing-claims.md) | Truthful marketing claims | Proposed |

## Everything that lands on a landing site

`LAND` holds what had no home before. A landing site is also governed by specs
in other groups, which stay where they are. The full checklist is:

| Area | Specs |
|---|---|
| Page structure & content | [LAND-001](LAND-001-landing-page-anatomy.md), [LAND-002](LAND-002-calls-to-action-and-store-links.md), [LAND-012](LAND-012-truthful-marketing-claims.md) |
| Header & footer | [UI-007](../design/UI-007-navigable-landing-header.md), [LAND-003](LAND-003-responsive-navigation.md), [UI-008](../design/UI-008-unified-footer.md) |
| Media | [UI-005](../design/UI-005-landing-page-screenshots.md), [LAND-006](LAND-006-showcase-media-pipeline.md), [DEL-004](../delivery/DEL-004-agent-operable-screen-capture.md) |
| Pricing | [LAND-005](LAND-005-pricing-section.md), [IAP-002](../../IAP-Subscriptions/specs/IAP-002-price-display-fidelity.md), [`general-specs-payment/plans/`](../../general-specs-payment/plans/plan-model.md) |
| Language | [I18N-001](../internationalization/I18N-001-localization.md), [LAND-004](LAND-004-language-and-locale-persistence.md) |
| Legal & support routes | [PAGE-001](../pages/PAGE-001-privacy-route.md) `/privacy`, [PAGE-002](../pages/PAGE-002-imprint-route.md) `/imprint`, [PAGE-003](../pages/PAGE-003-support-route.md) `/support`, [PAGE-005](../pages/PAGE-005-terms-route.md) `/terms`, [UI-006](../ui/UI-006-data-privacy-and-support-links.md) |
| Auth entry points | [PAGE-004](../pages/PAGE-004-password-reset-routes.md), [AUTH-002](../authentication/AUTH-002-no-tokens-in-urls.md), [AUTH-006](../authentication/AUTH-006-secure-web-to-app-handoff.md) |
| Trust pages | [LAND-007](LAND-007-security-overview-page.md) `/security`, [LAND-008](LAND-008-docs-and-setup-guides.md) `/docs` and guides |
| SEO | [SEO-001](../seo/SEO-001-robots-txt.md), [SEO-002](../seo/SEO-002-sitemap.md), [SEO-003](../seo/SEO-003-metadata-and-social-cards.md), [SEO-004](../seo/SEO-004-structured-data.md) |
| Shell & plumbing | [LAND-009](LAND-009-document-shell.md), [LAND-010](LAND-010-icons-manifest-and-well-known-files.md), [LAND-011](LAND-011-not-found-scanners-and-legacy-redirects.md) |
| Design & accessibility | [UI-001](../design/UI-001-responsive-layout.md), [UI-002](../design/UI-002-visual-design-language.md), [UI-003](../design/UI-003-iconography.md), [A11Y-001](../accessibility/A11Y-001-baseline.md) |
| Edge & security | [SEC-006](../security/SEC-006-edge-rate-limiting.md) (shared WAF), [SEC-003](../security/SEC-003-request-integrity-csrf.md) (forms) |

## What belongs here

A spec belongs in `LAND` when it constrains the **public marketing site as a
whole**: its sections, navigation, media, shell, or the files every public host
serves. A spec about one route that every app must ship (privacy, imprint,
support, terms) belongs in [`PAGE`](../pages/README.md). A presentational rule
that applies to all UI, in-app included, belongs in `UI`.

## WebhookPush conformance (2026-10-04)

The reference implementation is not fully conformant. This table collects the
gaps found while extracting these specs, so they can be tracked in the
WebhookPush ClickUp folder.

| Spec | Gap |
|---|---|
| LAND-001 §5 | The shared nav links bare `#how` / `#pricing`, which are dead on `/security`, `/ai` and `/docs` |
| LAND-001 §7 | The Showcase, Actions, AI, Integrations, Regions and Security sections are hard-coded English inside the localized `/` |
| LAND-002 §1–2 | The drawer's App Store link is `idREPLACE_ME`, while the hero has the real id |
| LAND-003 §1, §12 | Two hand-kept nav lists (desktop and drawer), and the drawer is unreachable without JS |
| LAND-004 §5, §7 | The locale cookie is not `Secure`, the redirect is 302, and English-only pages inherit the visitor's `lang` |
| LAND-005 §10 | Static plan prices and limits live in `i18n.ts`, not in the plan catalogue (see `marketing/RECONCILE.md` §1) |
| LAND-006 §4, §9 | Legacy screenshots shown as current; PNG only |
| LAND-007 §10 | No `/.well-known/security.txt` |
| LAND-008 §5 | `/docs` hard-codes the ingest CloudFront host |
| LAND-010 §4 | The manifest uses a combined `"any maskable"` SVG, with no 192 px or 512 px PNG |
| LAND-011 §3, §5 | The scanner regex 403s all `/.well-known/*` paths that aren't routed explicitly; the 404 page is English-only |
| UI-008 §1–2 | The landing footer has no Support link. `/security` has its own footer variant. `/datenschutz`, `/impressum` and `/terms` have no footer or nav |
| PAGE-001 §1 | The privacy notice is at `/datenschutz` (alias `/data-privacy`), with no `/privacy` |
| PAGE-002 §1 | The imprint is at `/impressum`, with no `/imprint` |
| PAGE-005 §7 | `/terms` has no "last updated" date |
| SEO-002 §5 | The sitemap is a hand-kept array |
