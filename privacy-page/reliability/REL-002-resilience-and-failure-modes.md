# REL-002 — Resilience & failure modes

- **Status:** Adopted
- **Group:** Reliability
- **Applies to:** All services.
- **Last updated:** 2026-06-16

## Requirement

1. Each external dependency call **MUST** have a defined failure mode and a
   timeout. The app **MUST NOT** hang indefinitely on a slow upstream.
2. Failure direction **MUST** be deliberate and documented:
   - Controls that protect **availability** (rate limiters, breach checks)
     **SHOULD** fail **open**.
   - Controls that protect **security/correctness** (auth, signature checks,
     PKCE) **MUST** fail **closed**.
3. Idempotency **MUST** hold where retries are possible: one-time codes are
   single-use; counters use atomic updates; handoffs can't double-apply.
4. User-facing flows **SHOULD** degrade gracefully (e.g. routing falls back to a
   straight line if the router is down) rather than erroring the whole action.
5. Hot-path changes **MUST** be rolled out dev → verify → prod, never edited
   straight on production (see [DEL-002](../delivery/DEL-002-environments-and-promotion.md)).

## Rationale

The hard part of serverless reliability is choosing *how* each thing fails. Making
fail-open vs fail-closed an explicit, reviewed decision prevents both self-inflicted
outages and silent security holes.

## Acceptance criteria

- [ ] Every outbound call has a timeout and a tested failure path.
- [ ] Rate limiter outage → requests allowed; auth/signature failure → request denied.
- [ ] Replaying a one-time code/handoff has no effect.
- [ ] A downed optional dependency degrades the feature, not the app.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `passwordBreached()` and `allow()` fail open; PKCE/CSRF/signature checks fail closed; `takeAuthCode` atomic single-use; BRouter snap falls back to straight points.
- **WebhookNotification:** atomic `consumeUsage` counter; documented fail directions; lesson learned: split dev→verify→prod for ingest hot-path edits.
