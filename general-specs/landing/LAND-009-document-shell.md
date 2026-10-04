# LAND-009 — Document shell: head, focus, motion & zoom

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** The root layout (`<html>`, `<head>`, global CSS) shared by every
  page of an app's public site.
- **Last updated:** 2026-10-04

The rules here live in one file, the root layout, and every page inherits them.
They are the small things that, missing, show up on every page at once: no focus
ring, a phone that zooms on every input, an unthemed dropdown on a dark page.

## Requirement

### `<head>`

1. Every document **MUST** emit the following:
   - `<meta charset="utf-8">`;
   - `<meta name="viewport" content="width=device-width, initial-scale=1">`, with
     **no** `maximum-scale` and **no** `user-scalable=no`
     ([UI-001](../design/UI-001-responsive-layout.md) §5);
   - the icon set and the manifest link ([LAND-010](LAND-010-icons-manifest-and-well-known-files.md));
   - `<meta name="theme-color">` matching the page background;
   - per-page metadata ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md)).
2. `<html lang>` **MUST** be the locale the page was rendered in
   ([LAND-004](LAND-004-language-and-locale-persistence.md) §9), and it **MUST**
   also be correct inside the root error boundary (falling back to the default
   locale).
3. **Declare the colour scheme.** A site with a fixed dark (or light) theme
   **MUST** declare it (`<meta name="color-scheme" content="dark">` plus
   `:root { color-scheme: dark }`), so browser-painted surfaces (select option
   lists, autofill, scrollbars, number spinners) match it. A site that follows
   the OS theme declares `light dark` and provides both palettes.
4. `<body>` **MUST** set the page background explicitly, so there is no white
   flash before the CSS loads.

### Global CSS (in the shell, not per component)

5. **One focus ring.** `:focus-visible` **MUST** have a visible outline (at least
   2 px, in an accent colour with 3:1 contrast against the background, with an
   offset). No component **MAY** set `outline: none` without a replacement
   ([A11Y-001](../accessibility/A11Y-001-baseline.md)).
6. **Reduced motion.** Under `prefers-reduced-motion: reduce`, animations and
   transitions **MUST** collapse to effectively zero, and `scroll-behavior`
   **MUST** fall back to `auto` ([UI-002](../design/UI-002-visual-design-language.md) §5).
7. **No zoom-on-focus.** On phones (≤ 640 px), `input`, `select` and `textarea`
   **MUST** render at 16 px or more, so iOS Safari doesn't zoom the layout on
   focus. Fix this through font size, never by disabling zoom (§1).
8. **Screen-reader utility.** A `.sr-only` class (visually hidden, still
   announced) **MUST** exist for labels that are visual-only elsewhere.
9. **Skip link** as the first focusable element, as specified in
   [LAND-003](LAND-003-responsive-navigation.md) §13.
10. **Contrast tokens.** Theme tokens for body text, secondary text and "faint"
    text **MUST** each meet 4.5:1 against every background they are used on
    (WCAG AA). The ratio **SHOULD** be noted next to the token, so a later colour
    tweak can be checked.

### Robustness

11. The not-found and error pages **MUST** render with inline styles, so they
    stand on their own if the app CSS fails to load
    ([LAND-011](LAND-011-not-found-scanners-and-legacy-redirects.md)).
12. Smooth scrolling for in-page anchors **MAY** be enabled (`scroll-behavior:
    smooth`), subject to §6.

## Rationale

Each item is a one-liner in the shell, and each one, when missing, is a
site-wide defect:

- **Focus ring.** Fields used to set `outline:none` with nothing in its place,
  and keyboard focus was invisible everywhere.
- **Zoom on focus.** iOS zoomed into every 14 px input, and the usual "fix",
  `maximum-scale=1`, takes pinch-zoom away from everyone who needs it.
- **Colour scheme.** A dark page with a light native dropdown looks broken.

Putting these in the root layout means a new page cannot forget them.

## Acceptance criteria

- [ ] The viewport meta has no `maximum-scale` or `user-scalable=no`, and pinch-zoom works on iOS.
- [ ] `<html lang>` matches the rendered locale, including on the error page.
- [ ] With a dark theme, an open `<select>` and browser autofill render dark.
- [ ] There is no white flash on a cold load with a throttled network.
- [ ] Tabbing through any page shows a visible focus ring on every interactive element, and grep finds no `outline: none` without a replacement.
- [ ] With reduced motion on, there are no transitions or smooth scroll.
- [ ] Focusing an input on an iPhone does not zoom the page.
- [ ] Every text token meets 4.5:1 on its backgrounds.

## Implementation notes

- **WebhookPush** is the reference. `app/root.tsx` covers the following:
  - viewport without `maximum-scale`;
  - favicon SVG + ICO, apple-touch-icon, manifest;
  - `theme-color` `#08090a` and `color-scheme: dark`;
  - one `:focus-visible` ring in `#d6ff3f`;
  - `.sr-only` and `.skip-link`;
  - the reduced-motion reset;
  - `@media (max-width:640px){input,select,textarea{font-size:16px!important}}`;
  - an explicit body background.
  `app/theme.ts` records the contrast fix next to `faint` (`#7a7f88`, at least
  4.5:1, previously `#5b6069` at 3.15:1).
