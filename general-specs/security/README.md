# Security (`SEC`)

Cross-cutting controls that protect every app from abuse and compromise,
independent of feature set. Authentication-specific rules live in
[`authentication/`](../authentication/); this group covers secrets, abuse
prevention, request integrity, transport/at-rest protection, enumeration,
edge/CDN abuse defense, the integrity of the third-party code we ship, and what
an attacker gets from holding the device.

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
| [SEC-009](SEC-009-dependency-and-build-supply-chain.md) | Dependency & build supply-chain integrity | Proposed |
| [SEC-010](SEC-010-sbom-and-vulnerability-response.md) | Software Bill of Materials (SBOM) & vulnerability response | Proposed |
| [SEC-011](SEC-011-build-provenance-and-attestation.md) | Build provenance & artifact attestation | Proposed |
| [SEC-012](SEC-012-apple-client-local-attack-hardening.md) | Apple client hardening against local attacks (the Swift run-through) | Proposed |
