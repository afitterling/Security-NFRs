# PRIV-001 — Data minimization & retention

- **Status:** Adopted
- **Group:** Privacy
- **Applies to:** All apps storing user or device data.
- **Last updated:** 2026-06-16

## Requirement

1. Apps **MUST** collect only the data needed for the feature at hand. Optional
   data (e.g. precise location) **MUST** be off by default and opt-in.
2. Data with a natural lifetime (share links, live positions, one-time codes,
   rate-limit counters, sessions) **MUST** carry a TTL and be auto-deleted; it
   **MUST NOT** accumulate indefinitely.
3. Location and other sensitive streams **MUST** be retained only as long as the
   sharing/feature is active, then expired.
4. Personal data **MUST NOT** appear in logs, analytics, or error reports beyond
   what is strictly necessary; tokens and secrets never appear there.
5. Third-party data egress (sending content to an external service) **MUST** be
   limited to what the feature requires and disclosed in the privacy notice.

## Rationale

Less data retained = smaller breach blast radius and simpler compliance. TTL-based
expiry is the cheapest durable way to honour minimization on serverless stores.

## Acceptance criteria

- [ ] Every table holding ephemeral/personal data has a TTL attribute that is set.
- [ ] Optional sensitive collection (location) defaults off.
- [ ] Logs contain no tokens, passwords, or unnecessary PII.

## Implementation notes

- **Emergency:** Shares/Positions tables TTL-expire (`expiresAt`); one-time login tokens TTL'd; location sharing opt-in (off by default).
- **OpenCycle / OpenOutdoor / Priorize / WebhookNotification:** auth codes, rate-limit counters, messages TTL-expire; webhook messages carry `expiresAt`.
- **All:** secrets/tokens excluded from logs (see [SEC-001](../security/SEC-001-secrets-management.md)).
