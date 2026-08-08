# PRIV-002 — GDPR/DSGVO user rights

- **Status:** Adopted
- **Group:** Privacy
- **Applies to:** All apps with EU-reachable users.
- **Last updated:** 2026-06-16

## Requirement

1. Each app **MUST** publish a privacy notice (Datenschutz) and, where legally
   required, an imprint (Impressum), reachable from the site/app.
2. Users **MUST** be able to **delete their account and associated personal
   data** (right to erasure). Deletion **MUST** remove or irreversibly anonymize
   the user's records, not just disable login.
3. Users **SHOULD** be able to obtain a copy of their data (portability/access).
4. A lawful basis (consent or legitimate interest) **MUST** exist for each data
   use; consent-based collection (e.g. location) **MUST** be revocable.
5. Transactional email **MUST** use a verified sender identity and **MUST NOT**
   be used for unsolicited marketing.

## Rationale

These are legal obligations for EU users, not nice-to-haves. Account deletion in
particular must actually erase data, which a serverless single-table design must
be built to support.

## Acceptance criteria

- [ ] Privacy/imprint pages exist and are linked.
- [ ] A user can trigger account deletion; afterwards their records are gone/anonymized.
- [ ] Consent-based collection can be turned off and stops further collection.

## Implementation notes

- **Emergency:** `datenschutz` / `impressum` routes present.
- **OpenCycle / OpenOutdoor:** server-side account deletion supported (`deleteAccount` in `cognito.ts`); marketing pages carry localized legal copy.
- **All:** transactional mail via SES verified identities (see [REL-001](../reliability/REL-001-observability-and-alerting.md) for delivery).
- **Gap to track:** confirm a self-service erasure entry point (UI) exists in every app, not just a backend capability.
