# SEO-002 — sitemap.xml

- **Status:** Proposed
- **Group:** SEO
- **Applies to:** Every public web surface (marketing site, web app, docs) that
  has more than one indexable URL.
- **Last updated:** 2026-07-18

## Requirement

1. Every public host **MUST** serve a valid XML sitemap that returns **HTTP 200**
   with `Content-Type: application/xml` (or `text/xml`) and conforms to the
   `sitemaps.org/schemas/sitemap/0.9` schema.
2. The sitemap **MUST** be referenced by a `Sitemap:` line in `robots.txt`
   (see [[SEO-001-robots-txt]] §2), and that referenced URL **MUST** resolve 200.
3. The sitemap **MUST** list **only canonical, indexable URLs** — absolute HTTPS,
   on the canonical production host. It **MUST NOT** list `noindex` pages,
   redirects, auth/token pages, or `?`-parameter duplicate views.
4. `<loc>` values **MUST** exactly match the page's `rel="canonical"` URL (same
   host, same trailing-slash policy), so the sitemap and canonical never disagree.
5. The sitemap **MUST** be generated from a single source of truth (the same list
   that drives canonical/routing) — **MUST NOT** be a hand-maintained list that
   can drift from the routes that actually exist.
6. Non-production hosts (preview/staging/dev) **MUST NOT** serve a crawlable
   sitemap: return **404** (and `X-Robots-Tag: noindex`), consistent with
   [[SEO-001-robots-txt]] §5.
7. `<lastmod>` **SHOULD** be included only when a real, accurate modification
   timestamp is available; a fabricated or build-time-`now` value is worse than
   omitting it. `changefreq`/`priority` **MAY** be omitted (largely ignored).

## Rationale

A sitemap tells crawlers exactly which URLs are canonical and worth indexing,
which matters most for small sites where internal linking is sparse. Listing only
indexable, canonical URLs — and keeping `<loc>` identical to `rel="canonical"` —
prevents the sitemap from contradicting the rest of the SEO surface, which is a
common cause of "discovered, not indexed". Generating it from the route list
keeps it from rotting as pages are added or removed.

## Acceptance criteria

- [ ] `GET /sitemap.xml` on the prod host returns 200 with an XML content type and valid `<urlset>`.
- [ ] `robots.txt` lists the sitemap URL and that URL resolves 200.
- [ ] Every `<loc>` is absolute HTTPS on the canonical host and matches that page's `rel="canonical"`.
- [ ] No `noindex`/auth/token/duplicate URL appears in the sitemap.
- [ ] Non-prod hosts return 404 for `/sitemap.xml`.
- [ ] The URL list is derived from code (routes/constants), not hand-maintained.

## Implementation notes

- Reference implementation (Ejectify landing, Remix): `app/routes/[sitemap.xml].tsx`
  builds `<urlset>` from `INDEXABLE_PATHS` in `app/lib/seo.ts` via `canonicalUrl()`
  — the *same* helper that produces `rel="canonical"`, so §4 holds by construction.
  Only `/` and `/support` are listed; `/imprint`, `/data-privacy`, and
  `/confirm/*` are `noindex` and intentionally excluded.
- Host-gating mirrors `[robots.txt].tsx`: a `PROD_HOST` check returns 404 on any
  other host (§6).
- For larger sites, generate the sitemap in the build (or split into a sitemap
  index) rather than per-request; the per-request route is fine at this scale.
- Related: [[SEO-001-robots-txt]], [[SEO-003-metadata-and-social-cards]].
