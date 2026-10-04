# LAND-003 — Responsive navigation & off-canvas drawer

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** The header navigation of each app's public site, on the landing
  page and every sub-page that renders it.
- **Last updated:** 2026-10-04

[UI-007](../design/UI-007-navigable-landing-header.md) requires that a header
with a home link exists. [UI-002](../design/UI-002-visual-design-language.md) §4
prefers off-canvas navigation. This spec is the behaviour contract for the nav
itself: what it contains, how it collapses, and the keyboard and screen-reader
plumbing a drawer needs to not be a trap.

## Requirement

### Content

1. **One nav component, one item list.** Desktop links and drawer entries
   **MUST** come from the same item list. A destination in one and missing from
   the other is drift.
2. **The nav carries:** the brand wordmark linking to `/`; the landing sections a
   visitor jumps to (how it works, pricing, …) as root-qualified anchors
   ([LAND-001](LAND-001-landing-page-anatomy.md) §5); the public sub-pages (docs,
   security, guides); the store link ([LAND-002](LAND-002-calls-to-action-and-store-links.md));
   the language switcher ([LAND-004](LAND-004-language-and-locale-persistence.md));
   **Log in**; and **Sign up**, styled as the one emphasized control.
3. It is a `<nav aria-label="Main">` containing real links. Sub-pages **MUST**
   render the same component, not a reduced copy.

### Collapse

4. Above the breakpoint (around 820 px, or wherever the inline links stop fitting
   on one line), the links render inline. Below it, they collapse behind a
   **menu button** that opens a drawer. The breakpoint **MUST** be chosen so the
   inline row never wraps or overflows at any width above it.
5. The menu button is a `<button>` with `aria-label="Open menu"` and an
   `aria-expanded` that reflects the state, and its target size is at least
   44 × 44 pt ([A11Y-001](../accessibility/A11Y-001-baseline.md)).

### The drawer

6. The drawer is a panel sliding in from the side, at most `min(86vw, 340px)`
   wide, over a dimmed overlay. It **MUST** be a dialog: `role="dialog"`,
   `aria-modal="true"` and an `aria-label`.
7. **While open**, the following hold:
   - body scroll is locked;
   - focus moves to the first focusable element in the panel;
   - Tab and Shift+Tab cycle inside the panel (focus trap);
   - **Escape** closes it, and so do a click on the overlay, the close button and
     activating any link inside it.
8. **On close**, focus returns to the menu button.
9. **While closed**, the panel **MUST** be removed from the tab order and the
   accessibility tree (`inert` plus `aria-hidden`), even though it stays mounted
   offscreen for the animation.
10. Each drawer entry **SHOULD** carry an outlined, single-colour icon
    ([UI-003](../design/UI-003-iconography.md)) and have a large touch target.
    The drawer repeats the sign-up CTA and the language switcher at the bottom.
11. The slide and fade transitions **MUST** honour `prefers-reduced-motion`
    ([LAND-009](LAND-009-document-shell.md) §6).
12. **Without JavaScript**, every destination is still reachable, for example
    through the footer or a `<noscript>` link list. The drawer is an enhancement
    of navigation, not its only form.

### Skip link

13. The first focusable element on every page **MUST** be a "Skip to content"
    link, visually hidden until focused. Its target **MUST** land *after* the nav,
    even when the nav is rendered inside `<main>`.

## Rationale

The drawer is the most common place a site goes from "looks fine" to
"inaccessible": a panel that is visually hidden but still in the tab order, focus
left behind on the page underneath, no way out with the keyboard. Each of §7–§9
is one line of code and one test, and missing any one of them is a real defect
for keyboard and screen-reader users. One item list (§1) prevents the quieter
failure, where the mobile menu and the desktop menu disagree about what the site
contains.

## Acceptance criteria

- [ ] Desktop links and drawer entries are rendered from one list. Adding an item adds it to both.
- [ ] Every sub-page renders the same nav, and every entry resolves from every page.
- [ ] At every width above the breakpoint, the inline links fit on one line. Below it, the menu button appears.
- [ ] The menu button exposes `aria-expanded`, and its target is at least 44 × 44 pt.
- [ ] Opening the drawer moves focus into it and locks body scroll. Tab cycles inside it.
- [ ] Escape, the overlay, the close button and every link close the drawer, and focus returns to the menu button.
- [ ] With the drawer closed, Tab never lands on a drawer element, and screen readers don't announce it.
- [ ] With reduced motion on, the drawer appears without a slide animation.
- [ ] "Skip to content" is the first tab stop and lands past the nav.

## Implementation notes

- **WebhookPush** is the reference: `components/LandingNav.tsx` with
  `components/useDrawer.ts`, which provides the scroll lock, Escape, focus-in,
  focus trap and focus return, plus `inertProps(open)` for the closed state. The
  breakpoint is 820 px through a scoped `<style>` block (the page is otherwise
  inline-styled). The skip-link target is `<div id="main">`, placed after the
  nav inside `<main>`.
- **Gap (§1):** desktop links and drawer items are two hand-written lists that
  have already diverged. "App Store" is only in the drawer, and the language
  switcher sits in different places.
- **Gap (§12):** with JavaScript disabled, the menu button does nothing on
  phones.
