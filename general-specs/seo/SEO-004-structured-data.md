# SEO-004 — Structured data (JSON-LD)

- **Status:** Proposed
- **Group:** SEO
- **Applies to:** Public pages whose content maps to a schema.org type eligible
  for a Google rich result (app/product pages, FAQ pages, articles, breadcrumbs).
- **Last updated:** 2026-07-18

## Requirement

1. Structured data **MUST** be emitted as **JSON-LD** in a
   `<script type="application/ld+json">` block. Microdata/RDFa **SHOULD NOT** be
   used for new pages.
2. Structured data **MUST** describe content that is **actually visible on the
   page**. Marking up hidden, absent, or misleading content is a policy violation
   and **MUST NOT** be done (e.g. no fake `aggregateRating`, no invented prices).
3. Type selection **MUST** match Google's supported rich-result features:
   - A downloadable app page **SHOULD** use `SoftwareApplication` with at least
     `name`, `operatingSystem`, `applicationCategory`, and an `offers` price
     (`"0"` for a free download).
   - A page with a real Q&A list the site owns **SHOULD** use `FAQPage` with
     `mainEntity` `Question`/`acceptedAnswer` pairs mirroring the on-page copy.
4. JSON-LD values **SHOULD** be localized to the rendered page (question/answer
   text, descriptions) so the markup matches what the visitor sees.
5. URLs inside JSON-LD (`url`, `downloadUrl`, `@id`) **MUST** be absolute HTTPS
   and consistent with the page's canonical (see [[SEO-003-metadata-and-social-cards]]).
6. Each page **SHOULD** emit only the schema types relevant to that page; do not
   attach `FAQPage` to pages without a visible FAQ, etc.
7. Emitted JSON-LD **MUST** be valid (parses, required properties present) and
   **SHOULD** pass Google's Rich Results Test / Schema Markup Validator without
   errors.

## Rationale

JSON-LD is the format Google recommends and the easiest to keep decoupled from
markup. Correct `SoftwareApplication` and `FAQPage` data can earn richer SERP
treatment (platform/price badges, expandable FAQ answers), improving visibility
and click-through at effectively zero ongoing cost. The hard rule is honesty:
Google penalizes structured data that describes content not present on the page,
so the markup is generated from the same localized source as the rendered copy.

## Acceptance criteria

- [ ] App/product page emits valid `SoftwareApplication` JSON-LD with name, OS, category, and an offer price.
- [ ] FAQ page emits `FAQPage` JSON-LD whose Q&A text matches the visible FAQ exactly.
- [ ] All JSON-LD describes on-page, visible content (no fabricated ratings/prices/answers).
- [ ] JSON-LD URLs are absolute HTTPS and consistent with canonical.
- [ ] Google Rich Results Test reports the intended feature(s) with no errors.

## Implementation notes

- Reference implementation (Ejectify landing, Remix): `app/lib/seo.ts` exports
  `softwareApplicationLd(appStoreUrl, description)` and `faqPageLd(faq)`, returned
  from `meta()` as Remix `{ "script:ld+json": {…} }` descriptors.
  - Home (`_index.tsx`): `SoftwareApplication`, `operatingSystem: "macOS"`,
    `offers.price: "0"` (free download; the Auto-Eject timer is a one-time IAP),
    `downloadUrl` = the App Store listing.
  - Support (`support.tsx`): `FAQPage` built from the same localized `t.support.faq`
    array that renders the on-page FAQ, so §2/§4 hold by construction.
- When adding a locale, the JSON-LD localizes automatically because it reads the
  active dictionary.
- Future candidates: `Organization`/`WebSite` sitewide, `BreadcrumbList` if a nav
  hierarchy is added.
- Related: [[SEO-003-metadata-and-social-cards]], [[SEO-002-sitemap]].
