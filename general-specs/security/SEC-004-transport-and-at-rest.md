# SEC-004 — Transport & at-rest protection

- **Status:** Adopted
- **Group:** Security
- **Applies to:** All apps and stored data. At-rest protection *on an Apple client
  device* is [SEC-012](SEC-012-apple-client-local-attack-hardening.md).
- **Last updated:** 2026-06-16

## Requirement

1. All traffic **MUST** be HTTPS/TLS. There **MUST** be no plaintext-HTTP path to
   any authenticated surface.
2. Cookies carrying sessions/tokens **MUST** be `Secure` + `HttpOnly` (see
   [AUTH-003](../authentication/AUTH-003-session-token-lifecycle.md)). `Secure`
   **MUST NOT** depend on a runtime flag like `NODE_ENV`.
3. Outbound calls that include a secret in the request **MUST** use a confidential
   channel and **MUST NOT** echo the secret back to clients or logs.
4. Data at rest **SHOULD** rely on the managed store's encryption (DynamoDB
   default encryption); secrets at rest live in the secret store, not plain tables.
5. Privacy-preserving lookups against third parties **SHOULD** use k-anonymity or
   equivalent so the full sensitive value never leaves the service (e.g. HIBP
   range API sends only a SHA-1 prefix).
6. Authorization for any per-resource access **MUST** be enforced server-side by
   the authenticated identity (e.g. partition data by user `sub`); the client
   **MUST NOT** be trusted to scope its own reads/writes.

## Rationale

TLS + hardened cookies stop network capture; server-side authorization stops
horizontal access (one user reading another's data); k-anonymity avoids leaking
secrets to the very services meant to protect them.

## Acceptance criteria

- [ ] No authenticated endpoint is reachable over plain HTTP.
- [ ] Session cookies are `Secure; HttpOnly` regardless of environment.
- [ ] A user cannot read/write another user's records by changing an id.
- [ ] Breach/secret lookups transmit only a non-reversible prefix/hash.

## Implementation notes

- **All apps:** Lambda Function URLs / CloudFront are HTTPS-only; DynamoDB encryption at rest by default.
- **OpenCycle / OpenOutdoor:** data partitioned by Cognito `sub`; API middleware resolves identity from a verified token, never from client input; HIBP k-anonymity in `passwordBreached()`.
- **Emergency:** session cookie `Secure: true` always; per-user items keyed by `userId`.
