# SEO-003 — Page metadata, canonical URLs & social cards

- **Status:** Proposed
- **Group:** SEO
- **Applies to:** Every indexable public page (marketing site, web app, docs).
- **Last updated:** 2026-07-18

## Requirement

1. Every indexable page **MUST** have a unique, descriptive `<title>` and a unique
   `<meta name="description">`. Titles **SHOULD** be ≤ ~60 chars and descriptions
   ~120–160 chars; both **MUST** be localized when the page has translations.
2. Every indexable page **MUST** emit a `<link rel="canonical">` with an
   **absolute HTTPS URL on the canonical production host**. Canonical **MUST NOT**
   point at a preview/dev host, and **MUST** apply a single trailing-slash policy
   consistently (root `/` kept; sub-paths without a trailing slash).
3. Canonical **MUST** be stable across query-string variants that don't change
   content (e.g. `?lang=`, tracking params): those variants canonicalize to the
   clean URL.
4. Every indexable page **MUST** provide Open Graph tags: `og:type`, `og:title`,
   `og:description`, `og:url` (= canonical), `og:image`, `og:site_name`, and
   `og:locale`. `og:title`/`og:description` **MAY** equal the page title/description.
5. Every indexable page **MUST** provide Twitter Card tags: `twitter:card`
   (`summary_large_image` when a wide image exists), `twitter:title`,
   `twitter:description`, `twitter:image`.
6. `og:image`/`twitter:image` **MUST** be an absolute HTTPS URL to a real asset.
   A purpose-built card (**1200×630**, ≤ ~300 KB) is **RECOMMENDED**; a
   representative product screenshot is acceptable as a fallback.
7. `og:locale` **MUST** reflect the locale actually rendered for the request
   (e.g. `en_US` / `de_DE`), and the document's `<html lang>` **MUST** match it.
8. Non-indexable pages (auth/token flows, legal boilerplate the site chooses not
   to index) **MUST** carry `<meta name="robots" content="noindex">` and **SHOULD**
   omit canonical/social tags rather than emit misleading ones.
9. **Multilingual URLs (`hreflang`):** when a site exposes the *same* content at
   *distinct* URLs per language, each **MUST** emit reciprocal
   `<link rel="alternate" hreflang="…">` tags (one per locale + `x-default`), each
   pointing at a 200-resolving URL. When a site serves multiple languages at a
   **single** URL (locale chosen by cookie/`Accept-Language`), `hreflang` is
   **not applicable** and **MUST NOT** be faked; making the alternate languages
   individually indexable first requires per-locale URLs (tracked as an open item).

## Rationale

Title/description/canonical are the core of how a page appears and de-duplicates
in search. Open Graph and Twitter Card tags control the link preview on Slack,
iMessage, LinkedIn, X, and Discord — without them a shared link is a bare URL,
which measurably depresses click-through. Pinning canonical/`og:url` to the
production host stops preview deployments and query-param variants from competing
with or outranking the real page. `hreflang` only helps when translations live at
separate URLs; emitting it for a single-URL, cookie-switched site would point
Google at non-existent language variants, so the spec forbids faking it.

## Acceptance criteria

- [ ] Each indexable page has a unique, localized title + meta description.
- [ ] Each indexable page emits `rel="canonical"` as an absolute HTTPS URL on the prod host, stable across `?lang=`/tracking params.
- [ ] `og:*` and `twitter:*` tags are present; `og:url` equals canonical; images are absolute HTTPS.
- [ ] `og:locale` and `<html lang>` match the rendered locale (verified for each supported locale).
- [ ] `noindex` pages carry the robots tag and don't emit canonical/social tags.
- [ ] Rich-result / social-card validators (Google Rich Results Test, opengraph.xyz) render the page cleanly.

## Implementation notes

- Reference implementation (Ejectify landing, Remix): `app/lib/seo.ts` exposes
  `SITE_ORIGIN`, `canonicalUrl()`, and `buildMeta({title, description, pathname,
  locale, noindex})`, which returns the full title/description/canonical/OG/Twitter
  set as Remix meta descriptors (canonical via `{ tagName: "link", rel: "canonical" }`).
  Routes call it from `meta()` using `location.pathname` and the root-loader locale.
- Canonical always uses `SITE_ORIGIN` (the prod host), never the request host, so
  CloudFront preview domains self-canonicalize to prod (they're also `noindex` via
  [[SEO-001-robots-txt]] §5). The root loader strips `?lang=` with a redirect, so
  §3 holds.
- `og:locale` is mapped from the resolved locale (`en`→`en_US`, `de`→`de_DE`); add
  a mapping entry when adding a locale.
- **Open item (Ejectify):** locale is cookie/`Accept-Language`-driven at a single
  URL, so only the default-language render is separately indexable and `hreflang`
  is intentionally absent (§9). To index German independently, introduce per-locale
  URLs (e.g. `/de/…`) and then add reciprocal `hreflang` + `x-default`.
- Related: [[SEO-002-sitemap]], [[SEO-004-structured-data]], [[I18N-001-localization]].
