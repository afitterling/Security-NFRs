# Sync (`SYNC`)

How a client's local data is reconciled with a server copy so the same account
sees the same state on every device — without one device silently destroying
another's edits, and without the user having to keep signing in. These
requirements constrain *behaviour* (merge semantics, cadence, durability,
visibility), not a specific storage vendor or transport.

| ID | Title | Status |
|----|-------|--------|
| [SYNC-001](SYNC-001-conflict-free-merge.md) | Per-record merge on sync (never clobber) | Adopted |
| [SYNC-002](SYNC-002-background-sync-cadence.md) | Sync cadence (launch + periodic) | Adopted |
| [SYNC-003](SYNC-003-durable-credentials.md) | Durable, device-only credentials (stay signed in) | Adopted |
| [SYNC-004](SYNC-004-sync-status-visibility.md) | Sync status & last-synced visibility | Adopted |
