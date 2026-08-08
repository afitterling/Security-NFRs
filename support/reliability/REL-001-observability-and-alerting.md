# REL-001 — Observability & alerting

- **Status:** Adopted
- **Group:** Reliability
- **Applies to:** All services.
- **Last updated:** 2026-06-16

## Requirement

1. Every function **MUST** emit structured logs for errors and significant events,
   with enough context to debug — but **MUST NOT** log secrets, tokens, or
   unnecessary PII (see [SEC-001](../security/SEC-001-secrets-management.md),
   [PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md)).
2. A health/readiness endpoint **SHOULD** exist for each public service.
3. Background jobs and critical paths (ingest, delivery, cron) **MUST** surface
   failures via an alert channel (email/push/ntfy/Pushover), not just logs.
4. Apps offering a "liveness" promise (e.g. dead-man's-switch monitoring)
   **MUST** detect and alert on a missed expected signal within the stated window.
5. Transactional email/push **MUST** be sent from a verified identity and
   failures **MUST** be logged and retried or surfaced.

## Rationale

Serverless hides servers, not failures. Without explicit alerting, a broken cron
or ingest path fails silently until a user complains.

## Acceptance criteria

- [ ] Errors are visible in logs with correlation/context and no secrets.
- [ ] A simulated job failure produces an alert on the configured channel.
- [ ] `/health` returns service status.
- [ ] A missed heartbeat (where applicable) alerts within the promised window.

## Implementation notes

- **WebhookNotification:** `monitor.ts` dead-man's-switch cron (5-min) alerting via push/ntfy/Pushover/email; `ingest.ts` failed-status handling + heartbeat; `alert.ts` fan-out.
- **OpenCycle / OpenOutdoor:** `/health` endpoint; SES for transactional/feedback mail.
- **Emergency:** `TrustChain` cron recompute every 20 min.
