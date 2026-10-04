# LAND-012 — Truthful marketing claims

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Every factual statement on an app's public site, store listing
  and launch copy: features, integrations, regions, numbers, security, pricing
  fine print.
- **Last updated:** 2026-10-04

## Requirement

1. **Every claim is true of the shipped production system** on the day it is
   published. A capability that exists only in a branch, a dev stage or a plan
   **MUST NOT** be described as available.
2. **Roadmap is labelled, every time.** Anything not yet live **MUST** carry an
   explicit status label next to it (`COMING SOON`, `SOON`, `BETA`), visually
   distinct from the live label (`WORKS TODAY`, `LIVE`). This applies to every
   grid, list or table that mixes live and planned items (integrations, regions,
   platforms). A planned item **MUST NOT** be shown without a label, and **MUST
   NOT** have a CTA that implies it works.
3. **Labels follow the release.** When an item ships, or is cancelled, its label
   **MUST** change in the same release. The live/planned state **SHOULD** come
   from one data list that the page renders, not from per-item hand-styled
   markup.
4. **Numbers come from their source.** Limits, quotas, retention, prices, locale
   counts and latency figures **MUST** be read from the system's own definitions
   (plan catalogue, generated price table, locale list) or explicitly marked as
   an example ("e.g. 142 ms"). A number retyped into copy is drift waiting to
   happen ([LAND-005](LAND-005-pricing-section.md) §10).
5. **No invented social proof.** There **MUST NOT** be any fake ratings, review
   counts, customer logos without permission, testimonials that weren't given,
   "trusted by N teams" without a source, or `aggregateRating` in structured data
   without real ratings ([SEO-004](../seo/SEO-004-structured-data.md) §2).
6. **Third-party names are factual.** Naming another product ("works with
   TradingView", "connects to Claude") **MUST** describe a real, working
   integration path, and **MUST NOT** imply partnership or endorsement that
   doesn't exist. A product's own logo is used only under its brand guidelines.
   Otherwise use its name in plain text.
7. **Fine print is a claim.** "No credit card", "cancel anytime", "free to
   start", "deletion means deletion" and "end-to-end" are each checked against
   the shipped flow ([LAND-002](LAND-002-calls-to-action-and-store-links.md) §6).
8. **Security and privacy claims** follow [LAND-007](LAND-007-security-overview-page.md)
   §4 and §6 (named mechanisms, auditable) and must match the privacy notice and
   the store privacy label ([PAGE-001](../pages/PAGE-001-privacy-route.md) §6–§10).
9. **Demo content is plausible and fictional.** Examples and mocks use invented
   data, and **MUST NOT** use real people, real customer events or real tickers
   presented as live results.
10. **Copy has one source.** Store listing text, landing copy and launch posts
    **SHOULD** come from one marketing source folder, with a **reconcile list**
    of known mismatches. That list **MUST** be empty before a store submission or
    a landing release.

## Rationale

Overstating a product on its own landing page is a consumer-protection issue
(UWG §5 misleading commercial practices, EU UCPD) and an App Review issue
(Guideline 2.3, accurate metadata) before it is a trust issue. It is also how
support load is created: every "coming soon" shown as live is a ticket from
someone who tried it. Labelling roadmap items is cheap, honest, and still sells
the direction. The reconcile list makes known drift visible, rather than leaving
it in someone's head until launch day.

## Acceptance criteria

- [ ] Every item in every integrations, regions or platforms grid carries a live or planned label, rendered from one data list.
- [ ] No planned item has a CTA that implies it works.
- [ ] Every limit, price and count on the page traces to the catalogue, the generated table or the locale list, or is marked as an example.
- [ ] No rating, review, logo or testimonial appears without a source and permission, and the JSON-LD has no `aggregateRating`.
- [ ] Each fine-print claim was verified against the shipped flow in the release checklist.
- [ ] The marketing reconcile list is empty at release.

## Implementation notes

- **WebhookPush** is the reference for §2: the integrations grid labels
  TradingView, GitHub/CI and Cron as `WORKS TODAY`, and n8n, Make and Zapier as
  `COMING SOON`, with a footnote that they already work through generic HTTP
  nodes. The regions grid marks Europe and US East `LIVE`, and Asia, Australia,
  South America and Africa `SOON`. `seo.ts` deliberately emits no
  `aggregateRating` (§5). `marketing/RECONCILE.md` is the reconcile list (§10).
- **Gaps:**
  - Both grids are inline arrays in `_index.tsx`, not shared with the app or
    with the region list the API actually supports (§3).
  - `RECONCILE.md` still lists open items: a price mismatch and placeholder
    URLs. The App Store placeholder is confirmed still in `LandingNav.tsx`
    (§10).
