# LAND-011 — Not-found pages, scanner traffic & legacy redirects

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Every public web host an app serves (marketing site, web app),
  for every URL that no route matches and every URL that used to exist.
- **Last updated:** 2026-10-04

## Requirement

### Unmatched URLs

1. **A real 404.** A URL that no route matches **MUST** return status `404`, not
   a soft 404 (`200` with "not found" copy), and not a redirect to `/`. It
   **MUST** carry `X-Robots-Tag: noindex, nofollow`.
2. **Every method, same answer.** `POST`, `PUT`, `PATCH` and `DELETE` to an
   unmatched path **MUST** get the same `404`/`403` as `GET`, never a framework
   `405` "no action for route" that reaches error alerting.
3. **Known scanner paths get `403`.** Paths that only vulnerability scanners
   request **SHOULD** be answered `403`, without being routed into the app, so the
   intent is visible in access logs:
   - dotfiles: `/.env`, `/.git/`, `/.aws/`;
   - `wp-*`, `phpmyadmin`, `cgi-bin`, `actuator`;
   - script and backup extensions: `.php`, `.asp`, `.sql`, `.bak`, `.zip`;
   - `config.json`, `secrets.yml` and similar.
   The list **MUST NOT** cover any path the app actually serves (`/.well-known/*`
   in particular — [LAND-010](LAND-010-icons-manifest-and-well-known-files.md)).
4. **Not-found is not an error.** 404s and scanner 403s **MUST NOT** be logged at
   error level or trigger alerts
   ([REL-001](../reliability/REL-001-observability-and-alerting.md)). Alerting on
   them trains everyone to ignore the alert channel. They **MAY** be counted as a
   metric.
5. **A human page.** The 404 body is a short, styled page: "Page not found", one
   sentence, and a link to `/`. It is rendered with inline styles so it stands
   alone if the CSS fails ([LAND-009](LAND-009-document-shell.md) §11). It
   **SHOULD** carry the header and footer when they render safely, and it
   **MUST** be localized when the locale is known.
6. **Edge first.** Floods of scanner traffic **SHOULD** be absorbed at the edge
   (WAF managed rules, [SEC-006](../security/SEC-006-edge-rate-limiting.md))
   before they reach a function. §3 is the backstop, not the wall.

### URLs that used to exist

7. **A shipped URL never dies.** Any URL that has appeared in a shipped app
   binary, the store metadata, an email, the docs or the sitemap **MUST** keep
   resolving for as long as any of those can still be in circulation. That is
   indefinitely for app binaries.
8. When content moves, the old path **MUST** redirect permanently: `301` for
   `GET`/`HEAD` and `308` for other methods (which preserves the method and
   body). A permanent redirect is one hop, straight to the final URL, never a
   chain.
9. **Legal and store URLs** (privacy, imprint, support, terms) **MUST NOT**
   change at all once filed with a store. If a better path is adopted, the filed
   path redirects to it permanently
   ([PAGE-001](../pages/PAGE-001-privacy-route.md) §1,
   [PAGE-002](../pages/PAGE-002-imprint-route.md) §1).
10. A **redirect register** (a comment block in the router config, or a table in
    the app's specs) **MUST** list each legacy path, its target, and why it can't
    be removed ("deep-linked from app builds ≤ 1.1").

## Rationale

A public host receives far more scanner traffic than human traffic. If every
`/wp-login.php` probe becomes a framework error, the alert channel fills with
noise, and the one real error gets missed. That happened: SST alert emails fired
for every unmatched probe and for a `POST` to a path with no action, until the
catch-all answered them like any other request.

The redirect rules protect the other direction. An app binary on someone's phone
still deep-links `/feedback` years after the page moved to `/support`, and a 404
there is a broken support button that no release can fix.

## Acceptance criteria

- [ ] `GET /does-not-exist` returns 404 with `X-Robots-Tag: noindex` and a styled page linking home.
- [ ] `POST /does-not-exist` returns 404, not 405.
- [ ] `GET /.env`, `/wp-login.php` and `/backup.sql` return 403, and `/.well-known/apple-app-site-association` still returns 200.
- [ ] A burst of 404s and 403s produces no error-level log and no alert.
- [ ] The 404 page renders correctly with the app CSS blocked.
- [ ] Every legacy path in the redirect register returns 301 (GET) or 308 (POST) to its final target in one hop.
- [ ] Every URL in the store listing, and every URL hard-coded in shipped app builds, resolves to a 200 (directly or through one redirect).

## Implementation notes

- **WebhookPush** is the reference:
  - `routes/$.tsx` throws a bare `Response` (so Remix's `handleError` stays
    silent), returning 403 for the `FORBIDDEN` regexes and 404 otherwise, with
    `X-Robots-Tag`.
  - `action = loader` covers non-GET methods, and its own `ErrorBoundary`
    renders inline-styled "Page not found".
  - `routes/feedback.tsx` gives `/feedback` → `/support` as 301 for the loader
    and 308 for the action, because shipped app builds open `/feedback`.
  - `vite.config.ts` comments act as the redirect register.
- **Gap (§5):** the 404 page is English-only and has no header or footer.
- **Watch (§3):** the first `FORBIDDEN` pattern, `/^\/\./`, matches every
  dot-path, including `/.well-known/*`. The AASA file survives only because it
  has an explicit route. A future `/.well-known/security.txt`
  ([LAND-007](LAND-007-security-overview-page.md) §10) gets a 403 unless it is
  routed too. Narrow the pattern to exclude `/.well-known/`.
