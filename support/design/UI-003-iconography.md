# UI-003 — Iconography (flat, monochrome, slim)

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** All user-facing surfaces (web + app).
- **Last updated:** 2026-06-16

## Requirement

1. Icons **MUST** be **flat** — single-weight line or solid glyphs. Skeuomorphic,
   3D, gradient-heavy, or illustrative "rich colour" / multicolour icons
   **MUST NOT** be used in the product UI.
2. Icons **SHOULD** inherit colour from the surrounding text/theme (`currentColor`)
   rather than carrying their own palette, so they adapt to light/dark and stay
   monochrome by default.
3. The app **SHOULD** draw from **one** icon set (consistent stroke width, grid,
   and corner style) — see the slim aesthetic in [UI-002](UI-002-visual-design-language.md).
   Mixing visually inconsistent sets **MUST NOT** happen.
4. Icons **MUST** be rendered as vectors (SVG / vector font / native vector
   assets), crisp at every density; raster PNG icons **SHOULD NOT** be used for UI glyphs.
5. Icon-only controls **MUST** carry an accessible name per
   [A11Y-001](../accessibility/A11Y-001-baseline.md); icons **MUST NOT** be the
   sole carrier of meaning where misreading has consequences.

## Rationale

A single flat, monochrome icon system is the cheapest way to keep the slim, modern
look coherent across web + native, theme automatically for dark mode, and avoid the
dated, heavy feel of multicolour icon packs.

## Acceptance criteria

- [ ] All UI icons come from one documented set with consistent stroke/grid.
- [ ] No multicolour / gradient / 3D icons appear in product screens.
- [ ] Icons recolour correctly in light and dark themes via `currentColor`.
- [ ] Icon-only buttons expose accessible names.

## Implementation notes

- Status `Proposed`: pick the canonical set per app (e.g. one line-icon library)
  and document it alongside the design tokens from [UI-002](UI-002-visual-design-language.md).
