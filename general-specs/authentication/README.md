# Authentication (`AUTH`)

How users prove who they are, and how sessions/tokens are issued, carried, and
ended. These requirements are independent of the auth provider (Cognito,
self-issued tokens, federated Apple) — they constrain *behaviour*, not vendor.

| ID | Title | Status |
|----|-------|--------|
| [AUTH-001](AUTH-001-password-credentials.md) | Password credentials | Adopted |
| [AUTH-002](AUTH-002-no-tokens-in-urls.md) | No tokens in URLs (one-time code + PKCE) | Adopted |
| [AUTH-003](AUTH-003-session-token-lifecycle.md) | Session & token lifecycle | Adopted |
| [AUTH-004](AUTH-004-account-recovery-and-verification.md) | Email verification & account recovery | Adopted |
| [AUTH-005](AUTH-005-federated-signin.md) | Federated sign-in (Apple) | Adopted |
| [AUTH-006](AUTH-006-secure-web-to-app-handoff.md) | Secure web→app token hand-in | Adopted |
