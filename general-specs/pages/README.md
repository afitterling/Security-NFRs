# Public pages & routes (`PAGE`)

The handful of **routes every app has to ship** regardless of what the app does:
the legal pages, the way to reach a human, and the way back into a locked-out
account.

| ID | Title | Status |
|----|-------|--------|
| [PAGE-001](PAGE-001-privacy-route.md) | The `/privacy` route | Proposed |
| [PAGE-002](PAGE-002-imprint-route.md) | The `/imprint` route (Impressum) | Proposed |
| [PAGE-003](PAGE-003-support-route.md) | The `/support` route | Proposed |
| [PAGE-004](PAGE-004-password-reset-routes.md) | The `/forgot` + `/reset` routes | Proposed |

## Why this group exists

Every other group here states a requirement that is **true of many surfaces** —
"forms are rate-limited", "pages are localized", "no tokens in URLs". These four
specs are the inverse: each is about **one route**, and says where it lives, that
it resolves, what it must contain, and which cross-cutting rules land on it.

Before this group, those requirements existed only as clauses scattered through
other specs — the footer must link an imprint ([UI-008](../design/UI-008-unified-footer.md) §2),
a support page must exist ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §5),
a forgot-password flow must exist ([AUTH-004](../authentication/AUTH-004-account-recovery-and-verification.md) §2).
Each said the route must be **linked**; none said what it must **be**. That gap
is exactly where these routes fail in practice: a reset flow with no link to it,
an imprint with a P.O. box, a `GET` confirm that a mail scanner triggers.

## What belongs here

A spec belongs in `PAGE` when it is about **a specific URL that every app
serves**. If it constrains behaviour that recurs across many routes, it belongs
in the group that owns that behaviour (`SEC`, `A11Y`, `I18N`, `SEO`, …) and gets
*referenced* from here.

Not every route is a `PAGE` spec: the landing page is governed by `UI-005` /
`UI-007`, the footer by `UI-008`, `robots.txt` and `sitemap.xml` by `SEO-001` /
`SEO-002`. Those are page-shaped but their requirements are presentational or
SEO-owned, and they already have homes.

## Shape

All four follow the same skeleton, so they can be reviewed against each other:

1. **Route & reachability** — the stable path, HTTPS/200, signed-out, no
   identifier in the URL, where it's linked from, one exported URL constant.
2. **Content / states** — what the page must say, or the addressable states the
   flow passes through.
3. **Rendering** — localized, responsive, accessible.
4. **Indexing** — one consistent posture across `robots.txt`, sitemap,
   canonical, and `<meta robots>`.
5. **Availability** — static where possible, alerted on, non-production copies
   `noindex`.

## Feature bundles

`PAGE-001` and `PAGE-003` each have an assembled build set — every applicable
spec copied into one folder:

- [`privacy-page/`](../../privacy-page/README.md) — the `/privacy` route
- [`support/`](../../support/README.md) — the support form end to end

`PAGE-004`'s mechanics live in [`double-opt-in-auth/`](../../double-opt-in-auth/README.md).
`PAGE-002` has no bundle: it is a static page with no endpoint behind it.

## Retired IDs

- **`PRIV-004`** — the original bundle-local draft of the `/privacy` route spec,
  in `privacy-page/route/`. Renumbered to [PAGE-001](PAGE-001-privacy-route.md)
  when this group was created. It never entered `INDEX.md`, so it was reserved
  but never canonical. Per the repo convention, the ID **MUST NOT** be reused.
