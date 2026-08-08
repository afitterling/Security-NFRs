# REL-004 — Client crash handling & recovery

- **Status:** Proposed
- **Group:** Reliability
- **Applies to:** All client apps with a component UI tree (mobile, desktop, web).
- **Last updated:** 2026-06-18

## Requirement

1. The UI tree **MUST** be wrapped in an error boundary so an unhandled render
   error shows a recovery screen — never a black, blank, or frozen screen.
2. The crash screen **MUST** tell the user plainly that the app crashed,
   reassure them about their data, and offer a recovery action (retry / reset).
3. The crash screen **MUST** surface diagnostics — app version, build, app/bundle
   id, environment, backend host, schema version — **and** the error message and
   stack, and **MUST** offer a one-tap way to send that report to a support
   channel (e.g. a pre-filled email).
4. The crash screen **MUST** be self-contained — it **MUST NOT** depend on the
   theme, store, or providers that may themselves have crashed (use a fixed
   palette and only platform/runtime values).
5. The diagnostics shown **MUST** come from a single shared source reused by the
   in-app diagnostics surface ([REL-005](REL-005-diagnostics-surface.md)), so the
   crash report and Settings always agree.

## Rationale

An unhandled error otherwise leaves the user staring at a frozen screen with no
recourse. A self-describing crash screen converts that into an actionable,
reproducible report that carries exactly the context needed to fix it.

## Acceptance criteria

- [ ] A thrown render error shows the crash screen, not a frozen/black screen.
- [ ] The screen states the app crashed and offers retry/reset.
- [ ] The report includes version/build/bundle id/env/schema + error + stack and a one-tap send.
- [ ] The crash screen renders even when the theme/store provider is the thing that crashed.

## Implementation notes

- **Nilo:** `src/ErrorBoundary.tsx` wraps the whole tree **outside** the theme/store providers, with a fixed palette. It shows "The app has crashed.", the shared `getDiagnostics()` rows plus the error/stack, a **"Try again"** reset, and an **Email the report** button (`mailto:info@sp33c.tech`, pre-filled subject/body). Diagnostics come from `src/diagnostics.ts`, shared with Settings ([REL-005](REL-005-diagnostics-surface.md)). Commit `93e94c6`.
