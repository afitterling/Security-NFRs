# SEC-005 — Account-enumeration resistance

- **Status:** Adopted
- **Group:** Security
- **Applies to:** Every app with accounts.
- **Last updated:** 2026-06-16

## Requirement

1. Login, sign-up, password-reset, and resend responses **MUST NOT** reveal
   whether a given email/phone is registered. Wrong-password and no-such-user
   **MUST** return the same message, status, and (within reason) timing.
2. Where the identity provider offers it, generic-error mode **MUST** be enabled
   (e.g. Cognito `preventUserExistenceErrors=ENABLED`).
3. Sign-up against an existing address **SHOULD** return a neutral message and
   spend comparable work to a fresh sign-up (no fast "already exists" tell).
4. Password-reset **MUST** always advance to the "code sent" state regardless of
   whether the address exists (see
   [AUTH-004](../authentication/AUTH-004-account-recovery-and-verification.md)).

## Rationale

Distinct error messages or timings let an attacker harvest valid accounts to
target with phishing or credential stuffing. Uniform responses close the oracle.

## Acceptance criteria

- [ ] Login with an unknown email and login with a wrong password are indistinguishable (body, status, timing).
- [ ] `/forgot` for unknown vs known emails is indistinguishable.
- [ ] Provider generic-error mode is on.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `cognitoMessage()` collapses `NotAuthorized`/`UserNotFound` to one message; `preventUserExistenceErrors=ENABLED`; neutral `/forgot` notice.
- **Priorize:** decoy-verify (`decoyVerify`) on unknown user + neutral sign-up message for timing/oracle parity.
- **WebhookNotification:** uniform auth errors in `lib/auth.ts`.
