# A11Y-001 — Accessibility baseline

- **Status:** Proposed
- **Group:** Accessibility
- **Applies to:** All user-facing surfaces (web + app).
- **Last updated:** 2026-06-16

## Requirement

1. Interactive web UI **SHOULD** meet **WCAG 2.1 AA**: sufficient colour contrast,
   visible focus states, and keyboard operability of every control.
2. Form inputs **MUST** have associated labels; icon-only controls **MUST** have
   accessible names (`aria-label` / `accessibilityLabel`).
3. App screens **SHOULD** support the platform screen reader (VoiceOver /
   TalkBack) and respect Dynamic Type / font-scaling without clipping.
4. Content **MUST NOT** rely on colour alone to convey meaning (errors, status).
5. Touch targets **SHOULD** be ≥ 44×44 pt.
6. Motion-heavy effects **SHOULD** honour "reduce motion".

## Rationale

Accessibility is both a legal/ethical baseline and a quality signal. Codifying it
once keeps every new screen from regressing.

## Acceptance criteria

- [ ] Keyboard-only users can complete sign-in and core flows; focus is always visible.
- [ ] Inputs and icon buttons expose accessible names.
- [ ] Error states are distinguishable without colour.
- [ ] Screen reader can navigate the primary task on each app.

## Implementation notes

- Status `Proposed`: adopt after a baseline audit of each app's primary flows.
- Auth pages already use labelled inputs and semantic structure (a good starting point); the dark-theme palettes need a contrast pass.
