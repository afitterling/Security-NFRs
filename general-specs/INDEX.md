# Index

Sparse map of every group folder and every spec. Full intro in [README](README.md).

### design/
- [UI-001](design/UI-001-responsive-layout.md) — Responsive & adaptive layout _(Proposed)_
- [UI-002](design/UI-002-visual-design-language.md) — Visual design language (modern, refined, slim) _(Proposed)_
- [UI-003](design/UI-003-iconography.md) — Iconography (flat, monochrome, slim) _(Proposed)_
- [UI-004](design/UI-004-usability-baseline.md) — Usability baseline _(Proposed)_
- [UI-005](design/UI-005-landing-page-screenshots.md) — Landing page must showcase product screenshots _(Proposed)_
- [UI-006](ui/UI-006-data-privacy-and-support-links.md) — Data-privacy link & support page _(Proposed)_
- [UI-007](design/UI-007-navigable-landing-header.md) — Landing page must have a navigable header (home link) _(Proposed)_
- [UI-008](design/UI-008-unified-footer.md) — Unified footer with legal & informational links _(Proposed)_

### authentication/
- [AUTH-001](authentication/AUTH-001-password-credentials.md) — Password credentials
- [AUTH-002](authentication/AUTH-002-no-tokens-in-urls.md) — No tokens in URLs (one-time code + PKCE)
- [AUTH-003](authentication/AUTH-003-session-token-lifecycle.md) — Session & token lifecycle
- [AUTH-004](authentication/AUTH-004-account-recovery-and-verification.md) — Email verification & account recovery
- [AUTH-005](authentication/AUTH-005-federated-signin.md) — Federated sign-in (Apple)
- [AUTH-006](authentication/AUTH-006-secure-web-to-app-handoff.md) — Secure web→app token hand-in

### sync/
- [SYNC-001](sync/SYNC-001-conflict-free-merge.md) — Per-record merge on sync (never clobber)
- [SYNC-002](sync/SYNC-002-background-sync-cadence.md) — Sync cadence (launch + periodic)
- [SYNC-003](sync/SYNC-003-durable-credentials.md) — Durable, device-only credentials (stay signed in)
- [SYNC-004](sync/SYNC-004-sync-status-visibility.md) — Sync status & last-synced visibility

### security/
- [SEC-001](security/SEC-001-secrets-management.md) — Secrets management
- [SEC-002](security/SEC-002-rate-limiting-and-lockout.md) — Rate limiting & lockout
- [SEC-003](security/SEC-003-request-integrity-csrf.md) — Request integrity & CSRF
- [SEC-004](security/SEC-004-transport-and-at-rest.md) — Transport & at-rest protection
- [SEC-005](security/SEC-005-account-enumeration.md) — Account-enumeration resistance
- [SEC-006](security/SEC-006-edge-rate-limiting.md) — Edge rate limiting & DDoS protection
- [SEC-007](security/SEC-007-infra-change-integrity.md) — Infrastructure change integrity (stack fingerprint)
- [SEC-008](security/SEC-008-resource-tagging.md) — Resource tagging

### privacy/
- [PRIV-001](privacy/PRIV-001-data-minimization-and-retention.md) — Data minimization & retention
- [PRIV-002](privacy/PRIV-002-gdpr-dsgvo-user-rights.md) — GDPR/DSGVO user rights
- [PRIV-003](privacy/PRIV-003-tracking-consent-att.md) — Tracking consent (App Tracking Transparency) _(Proposed)_

### internationalization/
- [I18N-001](internationalization/I18N-001-localization.md) — Localization & locale selection

### accessibility/
- [A11Y-001](accessibility/A11Y-001-baseline.md) — Accessibility baseline _(Proposed)_

### seo/
- [SEO-001](seo/SEO-001-robots-txt.md) — robots.txt _(Proposed)_
- [SEO-002](seo/SEO-002-sitemap.md) — sitemap.xml _(Proposed)_
- [SEO-003](seo/SEO-003-metadata-and-social-cards.md) — Page metadata, canonical URLs & social cards _(Proposed)_
- [SEO-004](seo/SEO-004-structured-data.md) — Structured data (JSON-LD) _(Proposed)_

### reliability/
- [REL-001](reliability/REL-001-observability-and-alerting.md) — Observability & alerting
- [REL-002](reliability/REL-002-resilience-and-failure-modes.md) — Resilience & failure modes
- [REL-003](reliability/REL-003-non-blocking-app-boot.md) — Non-blocking app boot _(Proposed)_
- [REL-004](reliability/REL-004-client-crash-handling.md) — Client crash handling & recovery _(Proposed)_
- [REL-005](reliability/REL-005-diagnostics-surface.md) — Diagnostics surface for support _(Proposed)_

### delivery/
- [DEL-001](delivery/DEL-001-infrastructure-as-code.md) — Infrastructure as code
- [DEL-002](delivery/DEL-002-environments-and-promotion.md) — Environments & promotion
- [DEL-003](delivery/DEL-003-per-environment-app-identity.md) — Per-environment app identity & builds _(Proposed)_

### data-and-api/
- [DATA-001](data-and-api/DATA-001-api-conventions.md) — API conventions
- [DATA-002](data-and-api/DATA-002-storage-conventions.md) — Storage conventions
- [DATA-003](data-and-api/DATA-003-schema-versioning-and-migration.md) — Schema versioning & forward migration _(Proposed)_
- [DATA-004](data-and-api/DATA-004-pagination.md) — Pagination _(Proposed)_
