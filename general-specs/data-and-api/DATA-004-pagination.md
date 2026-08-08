# DATA-004 — Pagination

- **Status:** Proposed
- **Group:** Data & API
- **Applies to:** Every JSON/HTTP API that returns an unbounded or user-growable
  collection (task lists, message history, ride/route lists, usage records).
- **Last updated:** 2026-06-19

## Requirement

1. List endpoints over an unbounded collection **MUST** be pageable and **MUST
   NOT** return the entire collection in a single response on a hot path. The
   server **MUST** cap the page size it will return regardless of the request.
2. Pagination **MUST** be **cursor-based**, not offset/`page`-based. The cursor
   **MUST** be an **opaque** token (e.g. base64url) the client only echoes back;
   clients **MUST NOT** parse or construct it.
3. The cursor **MUST** encode position only. The partition/owner **MUST** be
   rederived server-side from the authenticated identity on every page and
   **MUST NOT** be trusted from the cursor (see
   [DATA-002](DATA-002-storage-conventions.md) §1). A tampered or foreign cursor
   **MUST NOT** return another user's rows.
4. A paged response **MUST** use a consistent envelope carrying the page and a
   next-page token, e.g. `{ "items": [...], "nextCursor": "…" | null }`.
   `nextCursor` is non-null **iff** more items remain; an empty final page
   returns `null`. (Use the existing field name for the collection — e.g.
   `tasks` — if one is already established.)
5. Page-size selection (`limit`) **MUST** be validated as a bounded integer and
   return `400` with the standard `{ "error": "…" }` shape on a malformed value
   (see [DATA-001](DATA-001-api-conventions.md) §1–2). An out-of-range `limit`
   is clamped to the server cap or rejected, never honored unbounded.
6. An invalid/expired cursor **MUST** fail safely: either `400` with the standard
   error shape, or restart from the first page — never error 5xx and never leak
   internal key structure.
7. Paging **MUST** be served by a key/index query, not a table scan, and the
   server **MUST NOT** load the full collection just to slice one page (see
   [DATA-002](DATA-002-storage-conventions.md) §4, §6).
8. Adding pagination to a shipped endpoint **MUST** preserve backward
   compatibility: either keep the legacy unpaged response when no pagination
   params are sent, or version the endpoint. Existing sync clients (see
   [SYNC-001](../sync/SYNC-001-conflict-free-merge.md)) **MUST NOT** break.

## Rationale

Unbounded list responses are a latency, memory, and cost cliff that only appears
once a power user's collection grows — exactly when it hurts most. Cursor paging
maps directly onto DynamoDB's `LastEvaluatedKey`, stays correct under concurrent
inserts (unlike offset paging, which skips/duplicates rows when the set shifts),
and keeps each response small. Opaque, position-only cursors keep the wire
contract stable and uphold partition-by-identity as the authorization boundary.

## Acceptance criteria

- [ ] A list endpoint never returns more than its documented page cap in one
      response, even when asked for more.
- [ ] Walking `nextCursor` to exhaustion yields every item exactly once with no
      duplicates or gaps under steady state.
- [ ] A cursor minted for user A returns no rows (or `400`) when presented by
      user B.
- [ ] A malformed `limit` returns `400` + `{ "error": "…" }`; an invalid cursor
      returns `400` or restarts from page one — never `5xx`.
- [ ] The page query uses an index/key condition, not a scan, and does not read
      the whole collection per page.
- [ ] Existing clients that send no pagination params keep their prior behavior.

## Implementation notes

- **Priorize:** `GET /tasks` is the live candidate. Today it loops every
  `TASK#<id>` item under the user's partition and returns the full array. Target
  shape: opt-in paging via `?limit=&cursor=` returning `{ tasks, nextCursor }`,
  where the cursor is the next item's `sk` (base64url-encoded) and `pk` is always
  rederived from the session email; with no params it returns the full array as
  before (back-compat for iOS Sync v3). Legacy single-item-blob accounts return
  the whole blob on page one with `nextCursor: null` until the next `putTasks`
  migrates them per-item.
- **DynamoDB mapping:** `Limit` + `ExclusiveStartKey` on the `Query`; encode
  `LastEvaluatedKey` (position only) into the cursor; reconstruct it as
  `{ pk: userKey(email), sk }` on the next call so the partition is never taken
  from the client.
- **OpenCycle / OpenOutdoor:** ride/route lists partitioned by `sub` are the
  natural next adopters; project away large blobs (`points`) per
  [DATA-002](DATA-002-storage-conventions.md) §6 so a page stays small.
- **Lesson basis:** the per-item task model (one row per `TASK#<id>`) was adopted
  precisely to escape the 400 KB single-blob ceiling — pagination is the matching
  read-side bound so a large list is also cheap to fetch.
