# SYNC-005 — Local store as source of truth (platform-mirrored sync)

- **Status:** Proposed
- **Group:** Sync
- **Applies to:** Every client app whose data is kept in sync by a
  **platform-provided mirror** of a local database (SwiftData or Core Data with
  CloudKit, Room with a mirroring layer, …) rather than by an API of our own.
  Apps that push to our own backend are covered by
  [SYNC-001](SYNC-001-conflict-free-merge.md)–[SYNC-004](SYNC-004-sync-status-visibility.md);
  this spec adds what a platform mirror changes and what it does not.
- **Last updated:** 2026-09-06

## Feature

The **local database on each device is the source of truth**. Every read and
write in the app goes to that store and never to the network. Sync is a
**record-level mirror** of the store into the user's platform account, run by
the platform in the background: local saves are exported after the fact, remote
changes are imported into the same store and surface through the ordinary data
path. The app **MUST** be fully usable with no network, no account, and no
sync entitlement, with no visible difference other than the absence of other
devices' changes.

## Requirement

### Availability

1. **Offline first.** Every core operation **MUST** complete with no network and
   **MUST NOT** present the absence of a network as an error. Sync is additive.
2. **No account required.** With the device signed out of the platform account,
   or a build without the sync entitlement, the app **MUST** open the same local
   store and behave normally. It **MUST NOT** prompt, warn, or ask the user to
   sign in. Mirroring **MUST** begin on its own when an account appears, with no
   user action and no data loss in either direction.
3. **Never crash on the store.** Opening the persistent store **MUST NOT** be
   fatal. A corrupt file, failed migration or full disk **MUST** degrade to an
   empty in-memory session behind a visible banner that says the data is intact
   on disk and merely unopened. The degraded session **MUST NOT** be seeded with
   sample data, so it does not read as a factory-fresh install.
4. **Cold start from disk.** The first screen **MUST** render from the local
   store without waiting on the mirror. No spinner, placeholder or "syncing"
   gate may stand between launch and the user's data. See
   [REL-003](../reliability/REL-003-non-blocking-app-boot.md).

### Durability and integrity

5. **Single-save transactions.** A multi-record operation with an invariant
   (rollover, reorder, move) **MUST** be written in one save, so termination at
   any point leaves either the old state or the fully committed new one.
6. **Repair at launch.** Any invariant that spans more than one record **MUST**
   be re-established by an idempotent repair step that runs before the first
   view reads the data. This is also the reconciliation point for two devices
   that each satisfied the invariant locally and then merged; the repair
   **MUST** prefer the outcome that loses nothing.
7. **Stable identities.** Every entity **MUST** carry its own UUID. Store-assigned
   identifiers **MUST NOT** be persisted, exported, or used to link records: they
   differ per device and per import.
8. **Cascade in the schema.** Ownership deletes (a parent taking its children)
   **MUST** be expressed as relationship delete rules, not as code at each
   delete site.
9. **Snapshots in history.** Historical records **MUST** store the text they
   were created with, never a live reference, so a later rename or delete cannot
   rewrite the past.

### The mirror

10. **Private scope only.** The mirror **MUST** target the user's private
    database in their own platform account. No public or shared database, no
    server of ours, and no identity beyond the platform account.
11. **Convergence.** Two devices on one account **SHOULD** show the same data
    within one minute of a change while both are online and in the foreground.
    Background delivery **MAY** be deferred by the platform; the app **MUST**
    register for the platform's silent-push wake-up so it is not foreground-only.
12. **Mirror-compatible schema.** The schema **MUST** satisfy the mirror's
    constraints at the storage level (typically: every relationship optional,
    every attribute defaulted, no unique constraints), and **MUST** hide those
    constraints behind non-optional accessors so no caller sees them.
13. **Additive changes only after first ship.** Once a build with mirroring has
    shipped, a schema change **MUST** be a new optional attribute, a new entity
    or a new optional relationship. Renames **MUST** keep the on-disk name so an
    existing store migrates in place. Removing, retyping or requiring a field is
    a breaking change for every device still on the old build and **MUST NOT**
    ship without a migration plan. Complements
    [DATA-003](../data-and-api/DATA-003-schema-versioning-and-migration.md).
14. **Fail safe without the entitlement.** The mirror's container identifier
    **MUST** come from a build setting that is tied to signing, never a
    hard-coded string. A build signed without the entitlement **MUST NOT** ask
    for a mirrored configuration at all, because platform mirrors trap rather
    than error when the entitlement is missing.
15. **Conflict tolerance.** The mirror resolves conflicts per record and per
    field, last writer wins. The model **MUST** tolerate that: no field may
    encode a cross-device invariant that another field depends on, except one
    that (6) repairs.

### Backup

16. **A user-owned copy.** Export **MUST** write one self-describing file
    (format and version inside the file) that the user chooses where to keep,
    independent of the mirror. The raw database file **MUST NOT** be the backup.
17. **Import is additive and atomic.** Import **MUST NOT** replace or merge into
    existing data. A malformed, foreign, or future-major-version file **MUST**
    be rejected with the store untouched.
18. **Picker files are outside the sandbox.** A URL from the system file picker
    **MUST** be read under security-scoped access, or files in cloud storage
    fail silently on device.

### Privacy

19. **No third party sees the data.** The only network traffic carrying user data
    **MUST** be the platform mirror on the platform's own transport under the
    user's account. Any other endpoint the app calls **MUST NOT** receive
    records, history or identifiers. See
    [PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md).

## Relation to SYNC-001 – SYNC-004

| Spec | Under a platform mirror |
|---|---|
| SYNC-001 per-record merge | Provided by the platform; (15) is the app's remaining duty. |
| SYNC-002 cadence | Replaced by push-driven delivery; (11) sets the target. |
| SYNC-003 durable credentials | Not applicable: the account is held by the OS, the app holds nothing. |
| SYNC-004 status visibility | **Still applies.** The platform exposes little status; an app that shows none is carrying a gap, not an exemption. |

## Rationale

A platform mirror removes an entire backend, an account system and a merge
engine, but it also removes the app's ability to see or steer the sync. The
compensation is to make the local store unconditionally trustworthy: usable
offline, openable signed-out, never fatal, and repaired at launch. Everything
the app cannot control (delivery timing, conflict resolution) is then bounded
by invariants the app *can* control (single-save writes, repair, additive
schema). The entitlement rule exists because the failure mode when it is wrong
is a trap in a background queue seconds after launch, which no test catches and
no user can describe.

## Acceptance criteria

- [ ] In airplane mode, every core operation completes with no error or banner.
- [ ] Signed out of the platform account, the app opens the same data and shows no prompt; signing in starts mirroring without user action.
- [ ] With the store file replaced by garbage, the app launches to a banner and a usable empty session, and the file is left in place.
- [ ] First paint shows data from disk before any mirror activity.
- [ ] Killing the app mid-way through a multi-record operation leaves either the old or the new state, never a hybrid.
- [ ] Two devices that each performed the invariant-bearing operation offline converge, after sync and one launch, to a state satisfying the invariant with nothing lost.
- [ ] Two devices on one account see the same data within one minute while both are foregrounded.
- [ ] A build with the container setting empty launches and passes its tests; a signed release build opens the mirrored configuration.
- [ ] A schema diff that removes, retypes or requires a field is rejected in review without a migration plan.
- [ ] Export then import preserves the data; importing the same file twice appends; a foreign or malformed file changes nothing.

## Implementation notes

- **Shopping List (iOS + Mac Catalyst, SwiftData + CloudKit):** `ShoppingListApp.makeContainer` tries a `.private("iCloud.tech.sp33c.shoppinglist")` configuration first and falls back to `cloudKitDatabase: .none` on the same file; the container id comes from `Info.plist` `CloudKitContainer`, set from the `CLOUDKIT_CONTAINER` build setting (14). Store open never `fatalError`s: it degrades to an in-memory store and `RootView` shows the banner (3). `StoreIntegrity.repair` runs before any view reads `currentRunID` and restores one open run per list, adopting the newest open run and archiving nothing (6). Rollover is one `save()` (5). Relationships are stored optional with `originalName` and exposed as non-optional collections (12, 13). `Backup` writes a JSON file with `format`/`version`, imports additively, and reads picker URLs under security scope (16–18). `UIBackgroundModes: remote-notification` (11). Only CloudKit and the tips API leave the app; the tips body carries no notes, history or ids (19). Tests: `rolloverLeavesOneCurrentRun`, `rolloverIsIdempotentOnRetry`, `interruptedRolloverRecovers`, the backup round-trip and rejection tests. App spec: `shopping-list/specs/recurring-shopping-list.md` §8.5, §10, §12.2, §12.3. **Gap:** no sync status or last-synced indicator (SYNC-004).
