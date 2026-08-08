# UI-005 — Landing page must showcase product screenshots

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** Each app's public landing / marketing page (web).
- **Last updated:** 2026-06-16

## Requirement

1. The landing page **MUST** show real product **screenshots** (or short
   screen recordings) of the actual app — not only icons, abstract illustrations,
   or marketing copy.
2. Screenshots **SHOULD** appear above the fold or immediately below the hero, so a
   first-time visitor sees what the product looks like without scrolling far.
3. Screenshots **SHOULD** reflect the **current** UI (the design language of
   [UI-002](UI-002-visual-design-language.md)); stale screenshots that no longer
   match the app **MUST** be updated or removed.
4. Screenshot assets **MUST** be responsive and performance-budget-friendly:
   appropriately sized/compressed, lazy-loaded below the fold, and served with
   `width`/`height` (or aspect-ratio) to avoid layout shift.
5. Screenshots **MUST NOT** leak real user PII or secrets — use seeded/demo data,
   consistent with [PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md).
6. Each screenshot **SHOULD** have descriptive `alt` text per
   [A11Y-001](../accessibility/A11Y-001-baseline.md).

## Rationale

For these consumer apps the landing page is the primary conversion surface. Seeing
the real product builds trust and sets expectations far better than copy alone —
and tying screenshots to the live design language keeps marketing and product honest.

## Acceptance criteria

- [ ] Landing page shows ≥ 1 real product screenshot near the top.
- [ ] Screenshots match the current shipped UI.
- [ ] Images are sized/compressed, lazy-loaded, and reserve space (no CLS).
- [ ] No real PII/secrets in any screenshot; each has descriptive `alt` text.

## Implementation notes

- Status `Proposed`: adopt when each app's landing page is next revised.
- Coordinate screenshot refresh with any change to [UI-002](UI-002-visual-design-language.md)
  so marketing never lags the product look.
