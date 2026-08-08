# DATA-001 — API conventions

- **Status:** Adopted
- **Group:** Data & API
- **Applies to:** Every JSON/HTTP API.
- **Last updated:** 2026-06-16

## Requirement

1. APIs **MUST** use correct HTTP status codes: `200/201/204` success, `400`
   malformed, `401` unauthenticated, `403` unauthorized, `404` missing, `409`
   conflict, `415` wrong content type, `429` rate-limited, `5xx` server error.
2. Errors **MUST** return a consistent JSON shape (e.g. `{ "error": "…" }` or
   `{ "ok": false, "error": "…" }`) and **MUST NOT** leak stack traces, secrets,
   or internal identifiers.
3. State-changing endpoints **MUST** require and validate JSON content type and
   bounded, schema-checked bodies (see
   [SEC-003](../security/SEC-003-request-integrity-csrf.md)).
4. Authentication **MUST** be a verified bearer token or session; the identity is
   resolved server-side and used for authorization (see
   [SEC-004](../security/SEC-004-transport-and-at-rest.md)).
5. Endpoints **MUST** be idempotent where retried (see
   [REL-002](../reliability/REL-002-resilience-and-failure-modes.md)); one-time
   operations are single-use.
6. A normalized base URL **MUST** be used (no double slashes from a trailing-slash
   Function URL); path construction strips trailing slashes.

## Rationale

Consistent status/shape makes clients simple and errors debuggable without
leaking internals; the trailing-slash rule reflects a real bug already hit and
fixed.

## Acceptance criteria

- [ ] Each documented failure returns its specified status + JSON error shape.
- [ ] No 5xx body contains a stack trace or secret.
- [ ] A mutation without `application/json` returns 415.
- [ ] Base URLs never produce `…aws//path`.

## Implementation notes

- **All apps:** Hono/Remix handlers return typed JSON errors; bearer/session verified before handlers run.
- **Lesson:** Lambda URL trailing-slash bug (`…aws//messages` → 404) fixed by stripping the trailing slash before appending paths.
- **ClickUp API:** every list/task request is checked against the whitelisted folder; out-of-scope returns 403.
