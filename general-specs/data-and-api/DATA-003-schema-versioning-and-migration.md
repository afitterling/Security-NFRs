# DATA-003 — Schema versioning & forward migration

- **Status:** Proposed
- **Group:** Data & API
- **Applies to:** Any app that persists or syncs a structured user data model.
- **Last updated:** 2026-06-18

## Requirement

1. Any persisted or synced data model **MUST** carry an explicit schema version
   — e.g. a versioned envelope `{ v, items }`. A bare/legacy payload with no
   envelope **MUST** be detected as version 0.
2. On read, the payload **MUST** be migrated forward through an **ordered, pure**
   migration chain from its stored version up to the current version. An unknown
   **future** version (e.g. data written by a newer build) **MUST** be left
   untouched — never dropped or downgraded.
3. Each migration **MUST** be pure and defensive: it transforms version `n` to
   `n+1` against whatever that older version actually wrote. Every record
   **MUST** then pass through a normalize/coerce step that fills defaults.
4. The **same** envelope and migration path **MUST** be shared by every store of
   the model — local cache **and** any synced blob — so a payload pulled from
   another device upgrades identically to one read from disk.
5. After a forward migration runs, the upgraded payload **SHOULD** be persisted
   once (idempotently, non-blocking) so it isn't re-migrated each launch, and the
   migration event **SHOULD** be recorded for diagnostics (see
   [REL-005](../reliability/REL-005-diagnostics-surface.md)).
6. Evolving the model **MUST** bump the version and **append exactly one**
   migration. Versions are never reused or reordered.
7. Where the model is a **collection of records that merge per-record across
   devices** (see [SYNC-001](../sync/SYNC-001-conflict-free-merge.md)), **each
   record MUST also carry its own schema version**, stamped on every write to
   point at the current schema. Migration **MUST** then run **per-record**, each
   record upgrading from its **own** declared version (falling back to the
   envelope version, then 0, for legacy records that predate per-record
   versions). The envelope version remains as a fast, decrypt-free hint and a
   fallback; the per-record version is authoritative for migrating that record.

   **Rationale:** a synced set merged from two devices on different schema
   versions can legitimately hold records at mixed versions. A single
   whole-array version can't describe that set; a per-record version lets each
   record migrate forward independently so no record is mis-migrated or dropped.

## Rationale

One owned migration path means old on-disk data **and** an old blob synced from
another device both upgrade in place instead of being silently dropped. Purity +
a normalize pass keeps each migration responsible only for its own structural
change.

## Acceptance criteria

- [ ] Every persisted/synced model read goes through `readEnvelope → migrate → normalize`.
- [ ] A legacy (v0, un-enveloped) payload loads and upgrades without data loss.
- [ ] A blob written by another device at an older version upgrades on pull.
- [ ] A payload at an unknown newer version is left intact (no downgrade/loss).
- [ ] Adding a field is a single appended, pure migration + version bump.
- [ ] For a per-record merged model: every record carries its own `schemaVersion`, stamped to the current version on write.
- [ ] A set merged from two devices at different versions migrates **each** record from its own version (a mixed-version set upgrades record-by-record).

## Implementation notes

- **Nilo:** `src/migrations/index.ts` owns `TIMERS_SCHEMA_VERSION` and the `{ v, timers }` envelope. `readEnvelope` treats a bare array as v0. Migrations are **per-row**: `MIGRATIONS[n]` upgrades a single row from version `n`→`n+1`; `rowVersion` reads each row's own `schemaVersion` (falling back to the envelope version, then 0); `migrateRows` maps `migrateRow` over every row. `normalizeTimer` then coerces each row and **stamps `schemaVersion: TIMERS_SCHEMA_VERSION`** (preserving a higher version from a newer build rather than downgrading). Each `Timer` thus carries its own `schemaVersion` (`types.ts`), set on every write path (`normalizeTimer`, `seedTimers`, `addTimer`). The same path serves the local AsyncStorage store **and** the encrypted sync vault (`storage.ts`, `sync.ts`). A forward migration is persisted once (fire-and-forget) and recorded (`loadLastMigration`), surfaced in Settings › DETAILS. Mirrors [SYNC-001](../sync/SYNC-001-conflict-free-merge.md) (per-record merge) and [DATA-002](DATA-002-storage-conventions.md).
