# UI-002 — Visual design language (modern, refined, slim)

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** All user-facing surfaces (web + app) and marketing/landing pages.
- **Last updated:** 2026-06-16

## Requirement

1. Each app **MUST** ship a single, documented design language — type scale,
   colour palette, spacing scale, radii, elevation — applied consistently across
   every screen. Ad-hoc per-screen styling **MUST NOT** accumulate.
2. The visual style **SHOULD** be contemporary ("trendy") and high-quality:
   generous whitespace, clear typographic hierarchy, restrained palette,
   purposeful motion. It **SHOULD** read as deliberately designed, not as a
   default/unstyled framework theme.
3. Chrome **SHOULD** be **slim**: thin borders and dividers, low-weight surfaces,
   minimal nesting of cards/panels. Decoration **MUST NOT** crowd out content.
4. Primary and secondary navigation **SHOULD** use an **off-canvas** pattern
   (a slide-in panel / drawer that lives off-screen until invoked) rather than a
   permanently heavy sidebar, keeping the main canvas focused on content.
5. Motion **SHOULD** be subtle and consistent (short, eased transitions) and
   **MUST** honour "reduce motion" per [A11Y-001](../accessibility/A11Y-001-baseline.md).
6. "Beautiful" is a requirement, not a nicety: a screen that is functional but
   visibly unpolished (misaligned, inconsistent spacing, clashing weights) is a
   **defect** and **SHOULD** be tracked as one.

## Rationale

Visual quality is a product differentiator and a trust signal for these consumer
apps. Codifying one design language per app — and an explicitly slim, modern,
off-canvas-navigation aesthetic — stops every new screen from drifting and keeps
the "beautiful" bar testable rather than subjective.

## Acceptance criteria

- [ ] A design-language reference (tokens: type, colour, spacing, radius) exists and is the source of truth.
- [ ] New screens use design tokens; no hard-coded one-off colours/spacing in review.
- [ ] Navigation uses an off-canvas drawer on at least the small-viewport layout.
- [ ] Chrome is slim — thin dividers, minimal card nesting — per a design review checkpoint.
- [ ] A design reviewer signs off "looks polished" before a primary screen ships.

## Implementation notes

- Status `Proposed`: ratify once each app has a documented token set.
- Iconography is specified separately in [UI-003](UI-003-iconography.md) (flat,
  monochrome, non-rich-colour) to keep the slim aesthetic coherent.
- Landing-page presentation, including screenshots, is in [UI-005](UI-005-landing-page-screenshots.md).
