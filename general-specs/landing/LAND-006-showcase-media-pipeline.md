# LAND-006 — Showcase media pipeline & responsive images

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Every screenshot, device mock, composed image and social card on
  an app's public site.
- **Last updated:** 2026-10-04

[UI-005](../design/UI-005-landing-page-screenshots.md) requires real screenshots
near the top. This spec covers how those images are **produced, shipped and
served**: one pipeline from raw capture to page, art direction per viewport, and
the loading and accessibility details that keep them cheap and legible.

## Requirement

### Pipeline

1. **One script from capture to page.** Every product image on the site (phone
   frames, composed bands, the social card) **MUST** be produced by one
   checked-in script from raw simulator or device captures. It **MUST NOT** be
   hand-exported from a design tool. Re-running the script after a new capture
   updates every image.
2. **Captures are reproducible.** Raw captures **SHOULD** come from the
   agent-operable capture flow ([DEL-004](../delivery/DEL-004-agent-operable-screen-capture.md))
   against seeded demo data, so a UI change can be re-shot without manual
   staging.
3. **Demo data only.** No real user, email, token, webhook secret or
   identifiable content appears in any image or mock
   ([UI-005](../design/UI-005-landing-page-screenshots.md) §5).
4. **Stale UI is labelled or removed.** An image showing an older UI than the
   current release **MUST** be re-shot, or removed within one release
   ([UI-005](../design/UI-005-landing-page-screenshots.md) §3). It **MUST NOT**
   sit next to current screens as if it were current.

### Shipping

5. Images **MUST** be bundled through the build (fingerprinted, immutable,
   long-cached), not served from an ad-hoc route or a hot-linked external host.
   Sites with allow-listed routes **MUST NOT** depend on a route for an image.
6. **Mocks drawn in CSS** (a browser window, an iPad, a notification) are
   allowed and encouraged for UI that is not a screenshot. They **MUST** use the
   same sample data across devices, so the showcase tells one story, and they
   **MUST NOT** use real product trademarks they have no rights to.

### Serving

7. **Art direction per viewport.** When a composed image (e.g. three phones in
   one band) stops being legible below a width, the page **MUST** switch to
   individual frames at that width. Only the variant shown is downloaded: use
   `<picture>` / `srcset`, or `display:none` plus `loading="lazy"`, which
   browsers don't fetch.
8. Every image **MUST** carry intrinsic `width` and `height` (or an
   `aspect-ratio`) so it causes no layout shift. Images below the fold are
   `loading="lazy"` and `decoding="async"`. The hero image, if any, is **not**
   lazy.
9. **Budgets.** One screenshot **SHOULD** stay at or under about 250 KB, and a
   composed band under about 600 KB, served as WebP or AVIF with a PNG or JPEG
   fallback. The total image weight above the fold **SHOULD** stay under 1 MB.
10. **Alt text.** An informative image gets an `alt` that says what it shows. An
    image whose content is fully repeated by an adjacent caption, or a decorative
    duplicate of another image, gets `alt=""`
    ([A11Y-001](../accessibility/A11Y-001-baseline.md)). A composed band
    describes all of its screens in one `alt`.
11. **Edge fades, not hard cuts.** Composed renders that bleed past the content
    column **SHOULD** be masked at the edges (CSS `mask-image`), so the render's
    own background plate never draws a visible rectangle on the page.

### Social card

12. The `og:image` / `twitter:image` **MUST** be a purpose-built 1200 × 630 card
    from the same pipeline, carrying a real screenshot, with an `og:image:alt`
    ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §6). Its URL is
    absolute and **always** on the production host, never the requesting host.

## Rationale

Screenshots go stale silently. The app ships a new UI, and the landing page keeps
showing last quarter's. A pipeline that rebuilds every image from fresh captures
makes updating them a single command, rather than an afternoon in a design tool,
so it actually happens. The serving rules are about cost: a composed band that
looks great at 1240 px is a 2 MB thumbnail on a phone, so the phone gets the
individual frames and never downloads the band.

## Acceptance criteria

- [ ] One checked-in script regenerates every phone frame, composed band and the social card from raw captures.
- [ ] No image contains real user data.
- [ ] No image shows a UI older than the current release, unless it is labelled as such or scheduled for removal.
- [ ] Images ship as fingerprinted build assets.
- [ ] At 375 px, the network panel shows the individual frames and not the composed band. At 1280 px, the reverse.
- [ ] Every `<img>` has `width`/`height` or an aspect ratio. Lighthouse CLS is under 0.1.
- [ ] Below-the-fold images are lazy, and the image weight above the fold is under 1 MB.
- [ ] Informative images have descriptive `alt`, and decorative duplicates have `alt=""`.
- [ ] `og:image` is a 1200 × 630 card on the production host, with `og:image:alt`.

## Implementation notes

- **WebhookPush** is the reference:
  - `marketing/marketing-images/gen-web.sh` → `generate.py web` renders the
    phone frames, `trio-2400x1260.png` and `social-card-1200x630.png`, and
    copies them into `sst/landing/app/images/`.
  - Images are Vite-imported (fingerprinted) because the routes are a manual
    allow-list.
  - `.lp-trio` / `.lp-shot` switch at 760 px, and the hidden variant is
    `display:none` plus `loading="lazy"`.
  - `TRIO_MASK` fades the bottom and both sides.
  - `DeviceShowcase.tsx` draws the web, iPad and iPhone mocks in CSS from one
    shared sample data set.
- **Gap (§4):** three `phone-legacy-*.png` screenshots from the 2026-07-10 run
  ("older UI, no tags or archiving yet") are shown in their own row as current
  product screens.
- **Gap (§9):** all images ship as PNG. The trio is 2400 × 1260 with no WebP or
  AVIF variant.
