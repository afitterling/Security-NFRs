# SYNC-002 — Sync cadence (launch + periodic)

- **Status:** Adopted
- **Group:** Sync
- **Applies to:** Every app that keeps a local copy in step with a server copy
  while the app is in use.
- **Last updated:** 2026-06-17

## Feature

Once sync is **established** (the user is signed in and entitled to sync), the
app **MUST** reconcile automatically — both **on launch** and **periodically**
while in use — so a device picks up other devices' changes without the user
manually pressing "Sync now".

## Requirement

1. The app **MUST** sync once shortly after launch when a session is restored.
2. While sync is established, the app **MUST** re-sync on a recurring cadence in
   the **5–12 minute** range. The interval **SHOULD** be **jittered** (randomized
   within the range) rather than a fixed period, to avoid synchronized herds of
   clients hitting the backend at the same instant.
3. The periodic timer **MUST** be rescheduled **after** each run completes (not a
   fixed-rate interval that can stack), so a slow or failed sync cannot pile up
   overlapping requests.
4. The cadence **MUST** start/stop with the established state: it starts when the
   user becomes signed-in-and-entitled and is torn down on sign-out or loss of
   entitlement. It **MUST NOT** run (touch the network) when sync is not
   established.
5. A user-initiated **"Sync now"** **MUST** remain available and independent of
   the automatic cadence.
6. Automatic sync failures **MUST** be silent to the user beyond the normal
   status indicator (see [SYNC-004](SYNC-004-sync-status-visibility.md)); they
   **MUST NOT** interrupt with modal errors.

## Rationale

Manual-only sync means stale devices and surprising conflicts. A modest periodic
cadence keeps devices close to convergent at negligible cost, and jitter +
reschedule-after-completion keeps it gentle on the backend and immune to
request pile-up. 5–12 minutes is frequent enough to feel live without draining
battery or quota.

## Acceptance criteria

- [ ] Restoring a session on launch triggers exactly one sync.
- [ ] With sync established and the app foregrounded, a sync occurs every 5–12 min.
- [ ] Two intervals measured back-to-back are not identical (jitter present).
- [ ] Signing out stops the periodic sync (no further network calls).
- [ ] A sync that takes longer than the interval does not cause overlapping runs.
- [ ] "Sync now" works regardless of where the automatic timer is.

## Implementation notes

- **Tick:** `store.tsx` — a launch effect syncs once when `ready && auth`; a second effect, gated on `ready && auth && isPro`, runs a self-rescheduling `setTimeout` with `delay = (5 + random()*7) min`, cancelled on teardown. `syncNow()` no-ops when not entitled. Auto-sync errors surface only via `syncError`/status, never an alert.
