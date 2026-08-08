# UI-007 — Landing page must have a navigable header (home link)

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** Each app's public landing / marketing site and its sub-pages (web).
- **Last updated:** 2026-07-18

## Requirement

1. Every landing page and every one of its sub-pages (support, privacy, pricing,
   about, etc.) **MUST** render a persistent header containing a **home
   affordance** — a clickable brand logo/wordmark and/or a navigation menu — that
   returns the visitor to the landing page's home route (`/`).
2. From any sub-page, activating the home affordance **MUST** navigate to the
   home route in one interaction (e.g. from `/support` a click reaches `/`). The
   target **MUST** be the site root, not a scroll-to-top on the current page and
   not a dead `#` anchor.
3. The home affordance **MUST** be a real link (`<a href="/">` or the framework's
   `Link`), so it is keyboard-focusable, opens in a new tab on middle-click /
   ⌘-click, and is crawlable — it **MUST NOT** be a `div`/`span` with only a
   JavaScript `onClick`.
4. On the home route itself the header **MUST** still be present; the home
   affordance either links to `/` (no-op navigation) or scrolls to the top — it
   **MUST NOT** be missing.
5. The header **MUST** be reachable without signing in and **MUST NOT** be placed
   behind a paywall, account, or tracking-consent gate (consistent with
   [UI-006](../ui/UI-006-data-privacy-and-support-links.md)).
6. The logo/home link **MUST** have an accessible name (visible text, `alt`, or
   `aria-label`, e.g. "<App> — home") per
   [A11Y-001](../accessibility/A11Y-001-baseline.md), and the header **SHOULD**
   be consistent in placement and appearance across all pages
   ([UI-002](UI-002-visual-design-language.md)).

## Rationale

A clickable header that returns home is a universal web convention: visitors who
land deep (a support or privacy page from a store listing or search result)
expect to reach the product's home page by clicking the logo. Its absence is a
recurring, easily-missed gap — some pages ship it, others don't — that strands
users with no obvious way back. Pinning it as an NFR makes the omission catchable
in review instead of in support tickets.

## Acceptance criteria

- [ ] Every landing sub-page renders a header with a home logo/wordmark or menu.
- [ ] Clicking the header logo from any sub-page navigates to `/` in one click.
- [ ] The home affordance is a real anchor/link (focusable, ⌘-click opens a new tab), not an `onClick`-only element.
- [ ] The header is present on the home route too (not just sub-pages).
- [ ] The header is reachable signed-out, behind no paywall/consent gate.
- [ ] The logo/home link has an accessible name and consistent cross-page placement.

## Implementation notes

- Status `Proposed`: adopt when each app's landing site is next revised; audit
  existing pages for any that lack the header today and backfill.
- Complements [UI-005](UI-005-landing-page-screenshots.md) (landing content) and
  [UI-004](UI-004-usability-baseline.md) (clear path on every screen).
