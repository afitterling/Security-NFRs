# LAND-010 — Site icons, manifest & well-known files

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** The root of every public host an app serves (marketing site,
  web app), per stage.
- **Last updated:** 2026-10-04

Browsers, crawlers, operating systems and app stores all request a fixed set of
files at fixed paths, whether or not we serve them. Each one that is missing is
a 404 in the logs, a broken icon somewhere, or an OS feature that silently
doesn't work.

## Requirement

### The file set

1. Every public host **MUST** answer each of the following with `200` and the
   correct `Content-Type`:

   | Path | Content-Type | Purpose | Cache |
   |---|---|---|---|
   | `/favicon.svg` | `image/svg+xml` | Scalable tab icon | long, immutable |
   | `/favicon.ico` | `image/x-icon` | Bare request every browser makes; legacy browsers. Multi-resolution (16/32/48). | long, immutable |
   | `/apple-touch-icon.png` | `image/png` | iOS home screen, 180 × 180, opaque background | long |
   | `/site.webmanifest` | `application/manifest+json` | Install metadata | ~1 day |
   | `/robots.txt` | `text/plain` | [SEO-001](../seo/SEO-001-robots-txt.md) | ~1 day |
   | `/sitemap.xml` | `application/xml` | [SEO-002](../seo/SEO-002-sitemap.md) | ~1 day |
   | `/.well-known/security.txt` | `text/plain` | [LAND-007](LAND-007-security-overview-page.md) §10 | ~1 day |

   Apps with a native client add the following:

   | Path | Content-Type | Purpose |
   |---|---|---|
   | `/.well-known/apple-app-site-association` **and** `/apple-app-site-association` | `application/json` | Password AutoFill (`webcredentials`), universal links (`applinks`) |
   | `/.well-known/assetlinks.json` | `application/json` | Android App Links / credential sharing (Play apps) |
   | `/app-ads.txt` | `text/plain` | Required by ad networks; an app that sells **no** ads serves an intentionally empty file with a comment saying so |

2. **Explicitly routed.** Sites whose router uses an allow-list (no
   file-system routes) **MUST** declare each path above explicitly. A
   well-known file must not depend on a fallthrough.
3. **One icon source.** All icons **MUST** be generated from one master artwork
   (the app icon SVG) by a checked-in script, and they match the native app
   icon.

### Manifest

4. The manifest **MUST** carry `name`, `short_name`, `theme_color` and
   `background_color` (both equal to the page background,
   [LAND-009](LAND-009-document-shell.md) §1), `display`, and icons including a
   192 px and a 512 px PNG. A maskable icon **MUST** be a **separate** entry
   (`purpose: "maskable"`), with the artwork inside the safe zone. It **MUST
   NOT** be combined as `"any maskable"`, which crops the regular icon.

### Apple App Site Association

5. The AASA file **MUST** be served at both paths, as raw JSON, with **no
   redirect** (Apple's CDN does not follow redirects) and no file extension.
6. It **MUST** list the app ids for **every** build that claims the domain:
   `TEAMID.bundle` for production **and** for dev / per-stage builds
   ([DEL-003](../delivery/DEL-003-per-environment-app-identity.md)). The same
   file **MAY** be served on every stage's host, since an association only takes
   effect for an app that claims that domain. Leaving out the dev id makes
   Password AutoFill fail silently in dev builds.
7. A host that serves other `/.well-known/*` endpoints (OAuth metadata for an MCP
   or API) **MUST** route them by **specific** prefixes (`/.well-known/oauth-*`),
   never by a `/.well-known/*` catch-all that would swallow the AASA file.

### Per stage

8. Non-production hosts serve the same icons, manifest and AASA, but they follow
   the robots and sitemap rules for non-prod
   ([SEO-001](../seo/SEO-001-robots-txt.md) §5, [SEO-002](../seo/SEO-002-sitemap.md) §6).

## Rationale

`/favicon.ico` is requested by every browser on every first visit, even when the
page declares an SVG icon. Without a route it 404s, and on a framework with an
error-alerting hook, each 404 can also page someone. The AASA rules are all
production lessons:

- **Dev app id (§6).** Without it, AutoFill silently fails in dev builds.
- **No redirect (§5).** Apple's CDN does not follow one.
- **Specific prefixes (§7).** A `/.well-known/*` cache behaviour added for OAuth
  would have hidden the AASA file from iOS.

`app-ads.txt` exists, empty, to stop ad-network crawlers probing for it, and to
state publicly that the app sells no ad inventory.

## Acceptance criteria

- [ ] Each path in the table returns 200 with the listed `Content-Type` on the production host.
- [ ] `/favicon.ico` contains at least 16, 32 and 48 px images.
- [ ] Every icon comes from one script and one master artwork, and matches the native app icon.
- [ ] The manifest passes Lighthouse's installability check, with no combined `"any maskable"` icon.
- [ ] The AASA file returns JSON at both paths, without a redirect, and lists both the production and dev app ids. Apple's CDN validator (`app-site-association.cdn-apple.com/a/v1/<host>`) returns it.
- [ ] OAuth or other `/.well-known` endpoints don't shadow the AASA file.
- [ ] An app without ads serves an empty, commented `app-ads.txt`.

## Implementation notes

- **WebhookPush** is the reference: the routes `[favicon.svg]`,
  `[favicon.ico]` (multi-resolution, base64 in `icons/generated.ts`),
  `[apple-touch-icon.png]`, `[site.webmanifest]`, `[robots.txt]`,
  `[sitemap.xml]` and `[apple-app-site-association]` (both paths, `webcredentials`
  for `tech.sp33c.webhookpush` and `.dev`), plus `public/app-ads.txt` (empty,
  commented). Every route is declared in `vite.config.ts`. `sst.config.ts`
  routes only `/.well-known/oauth-*` to the MCP function, for exactly the reason
  in §7.
- **Gaps:**
  - The manifest uses one SVG with `purpose: "any maskable"` and no 192 px or
    512 px PNG (§4).
  - There is no `/.well-known/security.txt` (§1).
  - The sitemap is a hand-kept array in `[sitemap.xml].tsx`
    ([SEO-002](../seo/SEO-002-sitemap.md) §5).
