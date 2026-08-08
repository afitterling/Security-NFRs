# REL-003 — Non-blocking app boot

- **Status:** Proposed
- **Group:** Reliability
- **Applies to:** All client apps (mobile, desktop, web).
- **Last updated:** 2026-06-18

## Requirement

1. First paint **MUST NOT** block on network, disk, font/asset loading, or
   permission prompts. The app **MUST** render an interactive or loading UI
   within a bounded time regardless of slow or failed I/O.
2. Startup I/O (session restore, store load, remote config) **MUST** be
   time-bounded or fire-and-forget. A stuck or slow read **MUST NOT** freeze
   boot; the app boots with empty/default state and fills in when the read lands.
3. Font/asset loading **MUST** have a fallback — a timeout that proceeds with
   system fonts — so a failed or slow load never holds the splash screen.
4. Stale or invalid persisted state (dead sessions, a session minted against a
   now-different backend) **MUST** be detected and dropped on boot rather than
   hanging or erroring.
5. Permission prompts (notifications, tracking) **MUST NOT** be on the critical
   boot path; request them after the UI is interactive.

## Rationale

A boot that hangs on I/O is indistinguishable from a crash to the user. Bounding
every startup dependency keeps time-to-interactive predictable and turns a
"frozen splash" into, at worst, a brief empty state that self-heals.

## Acceptance criteria

- [ ] App reaches an interactive/loading state in airplane mode and with a slow disk.
- [ ] Font load has a timeout fallback; a blocked font never holds the splash.
- [ ] A stale/foreign session is dropped on cold boot, not hung on.
- [ ] No permission dialog blocks first paint.

## Implementation notes

- **Nilo:** font loading boots after `loaded || error || 2.5s timeout` (`App.tsx`); boot I/O is fire-and-forget and never blocks first paint; cold boot drops stale sessions and sessions minted against a different backend (`AuthState.backend`). Commits `3f20bdc`, `9ee6cf8`, `d1ab520`, `b46f126`. Permission prompts (ATT, notifications) run after the UI is up — see [PRIV-003](../privacy/PRIV-003-tracking-consent-att.md). Server-side failure modes are [REL-002](REL-002-resilience-and-failure-modes.md).
