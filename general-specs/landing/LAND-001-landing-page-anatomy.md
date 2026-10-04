# LAND-001 — Landing page anatomy & section anchors

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Each app's public landing / marketing page at `/` (web).
- **Last updated:** 2026-10-04

## Requirement

1. The landing page **MUST** answer three questions in order, before anything
   else: **what it is** (hero), **what it looks like** (product media,
   [UI-005](../design/UI-005-landing-page-screenshots.md)), and **what it costs**
   (pricing, [LAND-005](LAND-005-pricing-section.md)). Any other section sits
   between or after these and **MUST NOT** push them below a wall of feature copy.
2. The page **SHOULD** use this section order. Sections an app has no content for
   are omitted, never padded:

   | # | Section | Purpose |
   |---|---|---|
   | 1 | Header | Home link, nav, language, sign-in / sign-up ([LAND-003](LAND-003-responsive-navigation.md), [UI-007](../design/UI-007-navigable-landing-header.md)) |
   | 2 | Hero | Eyebrow, one-line value proposition, subtitle, primary CTAs ([LAND-002](LAND-002-calls-to-action-and-store-links.md)), one proof element |
   | 3 | Values | 3–4 short numbered value statements |
   | 4 | Showcase | Real screenshots / device mocks ([LAND-006](LAND-006-showcase-media-pipeline.md)) |
   | 5 | How it works | 3–4 numbered steps from zero to first result |
   | 6 | Feature deep-dives | One section per differentiating capability |
   | 7 | Integrations / platforms | What it works with, live vs. coming ([LAND-012](LAND-012-truthful-marketing-claims.md)) |
   | 8 | Security summary | Named controls, linking to `/security` ([LAND-007](LAND-007-security-overview-page.md)) |
   | 9 | Pricing | Plans, limits, localized prices ([LAND-005](LAND-005-pricing-section.md)) |
   | 10 | Final CTA | One heading, one button |
   | 11 | Footer | Unified footer ([UI-008](../design/UI-008-unified-footer.md)) |

3. **The hero has exactly one proof element.** This is something concrete that
   shows the product working: a terminal snippet with its real response, a
   notification mock, or a short screen recording. Prefer it to an abstract
   illustration. It **MUST** show real request/response shapes, with secrets
   masked.
4. **Stable section anchors.** Every section a nav entry or another page links to
   **MUST** carry a stable `id` (`#how`, `#pricing`, `#security`) and a
   `scroll-margin-top` that clears the header. Anchor ids are public URLs once
   linked from elsewhere (docs, store copy, emails) and **MUST NOT** be renamed
   without keeping the old id.
5. **Anchors work from every page.** A link to a landing section rendered on any
   other page **MUST** use the root-qualified form `/#pricing`, never a bare
   `#pricing`. A bare fragment on `/security` points at nothing.
6. **Headings form one outline.** There is one `<h1>` (the hero title), an `<h2>`
   per section, and an `<h3>` per card or step. Visual eyebrow labels are not
   headings ([A11Y-001](../accessibility/A11Y-001-baseline.md)).
7. **Every section is localized, or the page is not.** All copy on `/` **MUST**
   come from the locale catalog
   ([I18N-001](../internationalization/I18N-001-localization.md) §1). A section
   hard-coded in English inside an otherwise localized page is a defect, not an
   English-only page (see [LAND-004](LAND-004-language-and-locale-persistence.md) §7).
8. **It works without JavaScript.** The landing page **MUST** be server-rendered.
   Every link, anchor, CTA and the language switcher works with JS disabled. JS
   **MAY** enhance (drawer animation, copy buttons) but **MUST NOT** gate content.
9. **One page width.** Sections share one max content width (e.g. 1120 px) and
   one horizontal gutter, and they reflow to a single column on phones
   ([UI-001](../design/UI-001-responsive-layout.md)). No horizontal scroll at
   320 px.

## Rationale

A visitor decides in a few seconds whether to keep reading, and the three
questions in §1 are the ones they arrive with. A fixed section order also makes
every app's landing page reviewable against the same checklist, rather than each
one reinventing its structure. Anchors are the part that silently breaks: the nav
is shared across sub-pages, so a bare `#pricing` that works on `/` is a dead link
everywhere else.

## Acceptance criteria

- [ ] Hero, product media and pricing are all present, in that order.
- [ ] The hero has one value-proposition headline, primary CTAs and one concrete proof element with masked secrets.
- [ ] Every nav or cross-page section link resolves from `/security`, `/docs` and every other sub-page (no bare `#fragment` links in shared components).
- [ ] Each linked section has a stable `id` and a `scroll-margin-top`, and jumping to it does not hide its heading under the header.
- [ ] The page has exactly one `<h1>`, and the section headings form a valid outline.
- [ ] With `?lang=de` (or a German `Accept-Language`), no section renders in English.
- [ ] With JavaScript disabled, every link, CTA, anchor and the language switcher works.
- [ ] No horizontal scroll at 320 px.

## Implementation notes

- **WebhookPush** (`webhook/sst/landing/app/routes/_index.tsx`) is the reference.
  It follows the full table: hero with a `curl` proof card, values, a device
  showcase, "In action" screenshots, how it works, Actions, AI agents,
  Integrations, Regions, why limits, security summary, pricing, final CTA and
  footer.
- **Gap (§5):** `LandingNav` links `#how` and `#pricing` as bare fragments but is
  also rendered on `/security`, `/ai` and `/docs`, where those anchors don't
  exist. `Actions` and `Regions` already use `/#actions` and `/#regions`.
- **Gap (§7):** the Showcase, Actions, AI, Integrations, Regions and Security
  sections are hard-coded English inside a page whose hero, values, how-it-works
  and pricing are localized into 22 languages.
