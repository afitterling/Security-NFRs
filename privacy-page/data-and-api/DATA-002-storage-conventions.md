# DATA-002 — Storage conventions

- **Status:** Adopted
- **Group:** Data & API
- **Applies to:** Every app using DynamoDB (the default store).
- **Last updated:** 2026-06-16

## Requirement

1. Per-user data **MUST** be partitioned by the authenticated identity (Cognito
   `sub` / `userId`) so a query can only ever return the caller's items.
2. Ephemeral data (codes, counters, sessions, live positions, shares) **MUST**
   set a TTL attribute and rely on DynamoDB TTL for deletion (see
   [PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md)).
3. Counters (rate limits, usage) **MUST** use atomic `ADD`/`UPDATE`, never
   read-modify-write, to be correct under concurrency.
4. Single-table designs **MUST** document their key schema (pk/sk and any GSIs)
   in code; access patterns **MUST** be served by an index, not a scan, on hot
   paths.
5. Writes **MUST** strip undefined values (`removeUndefinedValues`) and **MUST
   NOT** persist secrets in plaintext attributes.
6. `marshallOptions`/projection **SHOULD** avoid loading large blobs when only a
   summary is needed (project away big arrays for list views).

## Rationale

Partition-by-identity is the cheapest durable authorization; TTL handles
retention; atomic counters keep limits/quotas correct under load. These mirror
patterns already in use across the apps.

## Acceptance criteria

- [ ] A user query cannot return another user's rows.
- [ ] Ephemeral tables have a set TTL attribute.
- [ ] Limit/usage counters use atomic updates.
- [ ] Hot-path reads use an index, not a table scan.

## Implementation notes

- **OpenCycle / OpenOutdoor:** rides/routes/locations partitioned by `sub`; `OpenCycleAuth`/`OpenOutdoorAuth` KV table with TTL for codes/counters/lockout; atomic `ADD` in `allow()`; ride list projects away `points`.
- **Emergency:** single-table (USER#/CONTACT#/TOKEN#/APIKEY#/PHONE#) with GSI1 by phone key; TTL on tokens/shares; documented key schema in `db.server.ts`.
- **WebhookNotification:** Users/Webhooks/Messages/Devices/Usage tables; atomic `consumeUsage`; messages TTL'd.
