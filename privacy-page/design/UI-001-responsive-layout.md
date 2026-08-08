# UI-001 — Responsive & adaptive layout

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** All user-facing surfaces (web + app).
- **Last updated:** 2026-06-16

## Requirement

1. Web UI **MUST** render correctly and remain fully usable from a 320 px narrow
   phone viewport up to large desktop widths, with no horizontal scrolling of the
   primary content and no clipped or overlapping controls.
2. Layout **MUST** be fluid/adaptive (relative units, flexbox/grid, container
   queries where helpful) rather than fixed-pixel — it **MUST NOT** depend on a
   small set of hard-coded device widths.
3. Primary navigation **SHOULD** collapse gracefully on small viewports (see the
   off-canvas pattern in [UI-002](UI-002-visual-design-language.md)) and expose
   the full feature set, not a reduced subset.
4. Tap/click targets and spacing **MUST** remain comfortable on touch (see the
   ≥ 44×44 pt target in [A11Y-001](../accessibility/A11Y-001-baseline.md)).
5. Content **SHOULD** reflow rather than zoom: text wraps, images scale, tables
   become scrollable or stacked. Pinch-zoom **MUST NOT** be disabled.
6. The app (Expo/native) **SHOULD** respect safe areas, notches, and both
   orientations where the screen supports them.

## Rationale

Most of these apps ship web + Expo native from one design. A layout that only
works at "desktop" or "phone" widths fails real users on tablets, split-screen,
and large phones. Fluid layout is cheaper to maintain than per-device breakpoints.

## Acceptance criteria

- [ ] Every primary screen is usable at 320 px wide with no horizontal scroll.
- [ ] No control is clipped, overlapped, or unreachable at any width 320 px–1920 px.
- [ ] Navigation is reachable and complete on small viewports.
- [ ] Zoom is not disabled; content reflows on zoom to 200%.

## Implementation notes

- Status `Proposed`: adopt after a responsive pass on each app's primary flows.
- Pairs with [A11Y-001](../accessibility/A11Y-001-baseline.md) (touch targets,
  reduce-motion) — responsiveness and accessibility are reviewed together.
