# AUTH-001 — Password credentials

- **Status:** Adopted
- **Group:** Authentication
- **Applies to:** Every app that accepts a user-chosen password.
- **Last updated:** 2026-06-16

## Requirement

1. Passwords **MUST** be at least **10 characters** and contain a lowercase
   letter, an uppercase letter, a number, and a symbol.
2. The server **MUST** enforce complexity itself, regardless of any client-side
   hint or identity-provider policy (defence in depth).
3. New or changed passwords **SHOULD** be checked against a known-breach corpus
   (e.g. Have I Been Pwned k-anonymity range API) and rejected if found. The
   check **MUST** fail open (allow) on provider error/timeout so an outage never
   blocks sign-up.
4. Passwords **MUST NOT** be stored or logged in plaintext. At rest they **MUST**
   be a salted, computationally-hard hash (scrypt/bcrypt/argon2) — or delegated
   to a provider that does the same (Cognito).
5. Password verification **MUST** be constant-time, and an unknown-user path
   **MUST** spend comparable time to a real verification (no timing oracle).
6. Where the provider supports it, server-side login **MUST** use a
   zero-knowledge exchange (SRP) so the raw password never transits the network;
   plaintext-password auth flows **MUST** be disabled.

## Rationale

A weak or breached password is the single most common account-takeover vector.
Enforcing complexity + breach screening at the server closes the gap left by
bypassable client checks. SRP and constant-time verification remove network- and
timing-based credential leaks.

## Acceptance criteria

- [ ] Submitting `Password1` (no symbol / too short) is rejected server-side.
- [ ] A password in the HIBP corpus is rejected; HIBP being unreachable still lets a strong password through.
- [ ] Stored credentials are salted hashes; no plaintext appears in logs.
- [ ] Login of a non-existent user takes ~the same wall-clock time as a wrong password.
- [ ] The identity provider's plaintext-password flow is disabled (SRP only).

## Implementation notes

- **OpenCycle / OpenOutdoor:** `passwordError()` + `passwordBreached()` in `config.ts`; Cognito pool policy mirrors it; client uses SRP (`amazon-cognito-identity-js`), `ALLOW_USER_PASSWORD_AUTH` removed.
- **Priorize:** `passwordError()` in `token.ts`; scrypt hashing; decoy-verify on unknown user for timing parity.
- **WebhookNotification:** `passwordOk` + scrypt in `lib/auth.ts`.
- **Emergency:** no user password (device key + handoff) — N/A for the credential parts; still subject to AUTH-002/003.
