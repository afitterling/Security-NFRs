# SYNC-004 — Sync status & last-synced visibility

- **Status:** Adopted
- **Group:** Sync
- **Applies to:** Every app that syncs user data and wants the user to trust that
  their data is safe across devices.
- **Last updated:** 2026-06-17

## Feature

The app **MUST** make the sync state legible at a glance: whether sync is
established, whether it is currently running, and **when it last succeeded**. The
user should never have to guess whether their data made it.

## Requirement

1. A persistent, lightweight **status indicator MUST** be visible from the main
   surface (e.g. an icon/badge in the header). Its state distinguishes at least:
   **established/idle** (e.g. green), **in-progress**, and **not established**
   (signed out or not entitled — e.g. muted).
2. The indicator **MUST** reflect *effective* sync, not just "signed in": it shows
   the established/green state only when sync will actually run (signed in **and**
   entitled **and** configured), so a signed-in-but-not-syncing account is not
   shown as safe.
3. The app **MUST** show a **"Last synced &lt;time&gt;"** value where the user
   manages sync, formatted in friendly relative terms ("just now", "5 min ago")
   and falling back to a clock/date for older syncs. The last-synced time **MUST**
   persist across launches.
4. The last-synced value **MUST** update immediately after a successful sync
   (including a manual "Sync now"), giving the user direct confirmation.
5. Status copy **SHOULD** stay terse and non-alarming; transient errors surface in
   the same indicator/line, not as interrupting dialogs (see
   [SYNC-002](SYNC-002-background-sync-cadence.md)).

## Rationale

Silent sync erodes trust: users keep a manual habit, or fear data loss, when they
can't see that it worked. A green dot plus an honest "last synced 2 min ago" is a
small, high-leverage signal — and gating "green" on *effective* sync avoids
falsely reassuring a user whose account can't actually sync.

## Acceptance criteria

- [ ] The header indicator is green/established only when signed in, entitled, and configured.
- [ ] The indicator shows an in-progress state while a sync runs.
- [ ] The sync screen shows "Last synced &lt;relative time&gt;" and it persists across relaunch.
- [ ] Tapping "Sync now" updates the last-synced value to "just now" on success.
- [ ] A failed sync shows a terse status, not a modal error.

## Implementation notes

- **Tick:** `store.tsx` tracks `lastSyncedAt` (persisted via `storage.saveLastSynced`, set on each successful `reconcile`). `TimersScreen` header cloud badge uses `willSync = auth && isPro && syncEnabled` → green (`c.ok`) idle / accent while `syncing` / faint otherwise. `AccountScreen` shows `Last synced ${formatLastSynced(lastSyncedAt)}` (relative → clock/date) next to "Sync now"; `format.ts:formatLastSynced`.
