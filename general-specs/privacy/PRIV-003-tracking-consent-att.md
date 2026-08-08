# PRIV-003 — Tracking consent (App Tracking Transparency)

- **Status:** Proposed
- **Group:** Privacy
- **Applies to:** Mobile apps that collect (or might collect) data used to track users.
- **Last updated:** 2026-06-18

## Requirement

1. An app that collects data **used to track** the user — linking it to
   third-party data for advertising, sharing it with data brokers, or collecting
   the advertising identifier (IDFA) — **MUST** request permission through the
   platform tracking-consent API (App Tracking Transparency on iOS) **before**
   any such data is collected.
2. Collection of tracking-only identifiers (IDFA, attribution ids) **MUST** be
   gated on an explicit **granted** result. On `denied`, `restricted`, or
   `undetermined` the app **MUST NOT** collect them and **MUST** remain fully
   functional without them.
3. The system prompt **MUST** be shown **once**, only while status is
   `undetermined`, with an honest usage string that states why.
4. Declining **MUST NOT** degrade core functionality, nag, or gate any feature
   behind consent.
5. The store **privacy label MUST match reality**: data is marked "used to
   track" **iff** the app actually tracks. An app that does **not** track
   **MUST NOT** present a tracking prompt and **MUST** mark the label as
   not-used-for-tracking.

## Rationale

Stores reject both directions of mismatch: a privacy label claiming tracking with
no prompt, and a prompt shown by an app that doesn't track. Gating the IDFA on an
explicit grant keeps the label truthful and the prompt meaningful, while letting
users who decline keep the full app.

## Acceptance criteria

- [ ] No tracking identifier is collected before consent is `granted`.
- [ ] The consent prompt appears exactly once (status `undetermined`), with a truthful usage string.
- [ ] The app is fully usable when tracking is denied — no feature gated, no repeat prompt.
- [ ] The store privacy label matches actual tracking behaviour.

## Implementation notes

- **Nilo:** `expo-tracking-transparency`; `requestTrackingOnce()` runs ~800ms after launch and only when status is `undetermined`. RevenueCat `collectDeviceIdentifiers()` (IDFA) is called **only** on `granted` (`src/tracking.ts` → `src/purchases.ts`). `NSUserTrackingUsageDescription` set via the config plugin. The App Store privacy label keeps Purchase History / Device ID / Product Interaction as tracking, consistent with the gated IDFA. See also [PRIV-001](PRIV-001-data-minimization-and-retention.md).
