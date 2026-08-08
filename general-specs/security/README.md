# Security (`SEC`)

Cross-cutting controls that protect every app from abuse and compromise,
independent of feature set. Authentication-specific rules live in
[`authentication/`](../authentication/); this group covers secrets, abuse
prevention, request integrity, transport/at-rest protection, enumeration, and
edge/CDN abuse defense.

| ID | Title | Status |
|----|-------|--------|
| [SEC-001](SEC-001-secrets-management.md) | Secrets management | Adopted |
| [SEC-002](SEC-002-rate-limiting-and-lockout.md) | Rate limiting & lockout | Adopted |
| [SEC-003](SEC-003-request-integrity-csrf.md) | Request integrity & CSRF | Adopted |
| [SEC-004](SEC-004-transport-and-at-rest.md) | Transport & at-rest protection | Adopted |
| [SEC-005](SEC-005-account-enumeration.md) | Account-enumeration resistance | Adopted |
| [SEC-006](SEC-006-edge-rate-limiting.md) | Edge rate limiting & DDoS protection | Adopted |
| [SEC-007](SEC-007-infra-change-integrity.md) | Infrastructure change integrity (stack fingerprint) | Adopted |
| [SEC-008](SEC-008-resource-tagging.md) | Resource tagging | Adopted |
