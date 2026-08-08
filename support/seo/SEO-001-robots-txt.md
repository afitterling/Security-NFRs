# SEO-001 — robots.txt

- **Status:** Proposed
- **Group:** SEO
- **Applies to:** Every public web surface (marketing site, web app, docs) served
  on its own host. Non-public/internal hosts are covered by rule 6.
- **Last updated:** 2026-06-20

## Requirement

1. Every public host **MUST** serve a `robots.txt` at the root
   (`https://<host>/robots.txt`) returning **HTTP 200** with
   `Content-Type: text/plain`. A missing file (404) is **NOT** an acceptable
   "allow all" — the file **MUST** exist and be intentional.
2. The file **MUST** declare a canonical `Sitemap:` line pointing to the absolute
   HTTPS sitemap URL (see [[SEO-002-sitemap]]). Multiple sitemaps **MAY** be
   listed.
3. Crawl rules **MUST** be explicit per user-agent. Production **SHOULD** default
   to `User-agent: *` / `Allow: /`, and **MUST** `Disallow:` paths that are
   non-indexable: auth flows, account/settings, API endpoints, search-result and
   filter URLs, cart/checkout, and any `?`-parameter duplicate views.
4. `robots.txt` **MUST NOT** be used as a security or privacy control. Paths that
   must stay private **MUST** be protected by auth/`noindex`, not merely
   `Disallow`-ed — listing them leaks their existence.
5. Non-production environments (preview, staging, dev hosts) **MUST** serve
   `User-agent: *` / `Disallow: /` and **MUST** also carry an
   `X-Robots-Tag: noindex` header, so non-prod content is never indexed.
6. Internal-only hosts (admin, internal APIs) **SHOULD** be unreachable by
   crawlers at the network layer; where publicly resolvable they **MUST**
   `Disallow: /`.
7. The file **MUST** stay small and valid (UTF-8, no BOM, ≤ a few KB). It
   **MUST** be generated/served from version control or a deterministic route —
   **MUST NOT** be hand-edited live on a server.
8. `Disallow` rules **MUST NOT** block static assets (CSS/JS/images) that the
   page needs to render, so crawlers can render and evaluate the page.

## Rationale

`robots.txt` is the first file a crawler fetches; a wrong one silently de-indexes
the whole site or, worse, exposes private paths or lets preview environments rank
in search. Making it explicit, version-controlled, and environment-aware keeps
crawl budget on the right pages and keeps non-prod and private surfaces out of
the index. It is a crawl directive, **not** an access control — hence rule 4.

## Acceptance criteria

- [ ] `GET /robots.txt` on every prod host returns 200 `text/plain`.
- [ ] The file lists at least one absolute HTTPS `Sitemap:` URL that itself resolves 200.
- [ ] Auth, account, API, and parameterized duplicate paths are `Disallow`-ed in prod.
- [ ] Preview/staging hosts return `Disallow: /` **and** `X-Robots-Tag: noindex`.
- [ ] No path that requires privacy relies on `Disallow` alone (auth/`noindex` present).
- [ ] CSS/JS/image asset paths needed for rendering are not blocked.
- [ ] The served file matches the version-controlled source.

## Implementation notes

- Status `Proposed`: adopt per web surface as each goes public.
- Prefer a framework route (e.g. Next.js `app/robots.ts` / a static `public/robots.txt`)
  so the file is built from code and environment-switched off `NODE_ENV` / deploy env,
  rather than a server-managed static file.
- Gate the prod vs non-prod variant on the deploy environment, and pair rule 5 with
  the platform's `noindex` header config (e.g. preview deployments).
- Related: [[SEO-002-sitemap]], [[SEC-006-edge-rate-limiting]] (bots/crawl floods are
  a rate-limiting concern, not a robots.txt one).
