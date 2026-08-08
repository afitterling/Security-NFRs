# UI-004 — Usability baseline

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** All user-facing surfaces (web + app).
- **Last updated:** 2026-06-16

## Requirement

1. Every primary task **MUST** have an obvious primary action and a clear path to
   completion; the user **SHOULD NOT** have to guess what to do next on any screen.
2. The UI **MUST** give feedback for every meaningful interaction: loading,
   success, and error states are explicit — no silent failures and no indefinite
   spinners without timeout/error handling.
3. Error messages **MUST** be human-readable and actionable (what went wrong +
   what to do), **MUST NOT** expose raw stack traces or internal codes, and
   **SHOULD** preserve the user's input on failure.
4. Destructive actions **MUST** be confirmable or reversible (confirm step or undo).
5. Empty states **SHOULD** be designed (explain the screen + offer the first
   action), not blank.
6. Forms **SHOULD** validate inline with clear messaging, use correct input types
   /keyboards, and minimise required fields (consistent with the data-minimisation
   stance of [PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md)).
7. Latency-sensitive actions **SHOULD** feel responsive (optimistic UI or a
   visible progress indicator within ~100 ms of the interaction).

## Rationale

"Good usability" is otherwise untestable. This pins it to concrete, reviewable
behaviours — feedback, error recovery, safe destructive actions, designed empty
states — so usability regressions are catchable in review, not just in support
tickets.

## Acceptance criteria

- [ ] Each primary screen has one obvious primary action.
- [ ] Loading / success / error states exist for every async action.
- [ ] Errors are actionable, input-preserving, and free of raw internals.
- [ ] Destructive actions are confirmable or reversible.
- [ ] Empty states are designed, not blank.

## Implementation notes

- Status `Proposed`: adopt after a usability pass on each app's core flows.
- Complements [A11Y-001](../accessibility/A11Y-001-baseline.md) (operability) and
  [UI-001](UI-001-responsive-layout.md) (works at every size).
