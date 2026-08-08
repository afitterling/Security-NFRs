# SYNC-001 — Per-record merge on sync (never clobber)

- **Status:** Adopted
- **Group:** Sync
- **Applies to:** Every app that syncs a user's records to a server copy shared
  across devices (timers, notes, settings, library items, …), especially when
  the server stores the set as one opaque/encrypted blob the client overwrites.
- **Last updated:** 2026-06-17

## Feature

When a device sends its data **upward**, it **MUST** merge with the current
server copy **per individual record**, and **MUST NOT** overwrite the whole
remote set with a stale local snapshot. A change made on another device that the
syncing device hasn't seen yet **MUST** survive the sync.

## Requirement

1. Merge is **keyed by stable record `id`** with **last-write-wins** on a
   monotonic `updatedAt` (per record), not a wholesale replace of the collection.
2. Every upward sync **MUST** re-pull the latest remote copy and merge it
   **immediately before** the write, so the window in which a concurrent edit can
   be lost is minimal. (Pull → merge → push, back-to-back; do not pull early, do
   slow work, then push a snapshot.)
3. **Deletes propagate via tombstones**, not by omission: a deleted record is a
   record with a `deleted` flag and a newer `updatedAt`, so the deletion wins the
   merge and reaches other devices instead of being resurrected by a device that
   still has the row.
4. The merge **MUST** be deterministic and commutative enough that two devices
   converging on the same inputs produce the same result regardless of order.
5. Where the backend stores the set as a single opaque/encrypted blob (the client
   cannot merge server-side), the client **MUST** still perform (1)–(3) on every
   push. The residual cross-device race (two devices writing between one's
   pull and push) **MUST** be closed with a **server-side conditional write**
   (compare-and-swap on a version/ETag, retry-on-conflict) — tracked as a gap
   until the storage endpoint supports it.

## Rationale

A naive "push my local list" overwrites the blob and silently drops edits made
elsewhere — the classic lost-update bug, and the one users notice fastest
(a timer they added on their iPad vanishes when their phone syncs). Per-record
last-write-wins with tombstones is the minimum that makes multi-device sync
trustworthy; a version-guarded write is what makes it correct under true
concurrency.

## Acceptance criteria

- [ ] A record created on device B and not yet seen by device A survives A's next push.
- [ ] Editing different records on two devices, then syncing both, keeps both edits.
- [ ] Editing the *same* record on two devices resolves to the newer `updatedAt`, on both devices.
- [ ] Deleting a record on one device removes it on the other after sync (no resurrection).
- [ ] No upward sync replaces the remote set wholesale from a snapshot older than the remote.
- [ ] (When supported) a push against a stale version is rejected and retried after re-merge.

## Implementation notes

- **Tick:** `sync.ts` — `mergeTimers(local, remote)` is id-keyed LWW on `updatedAt`, tombstones (`deleted:true`) win when newer; `pushMerged()` re-pulls and merges right before `putVault`, and `store.reconcile()` adopts the returned union. **Gap:** `/vault` is a whole-blob `PUT` with no version/ETag, so the cross-device race is minimized, not eliminated — add `If-Match`/version + retry on the endpoint to fully close it.
