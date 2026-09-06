# AUTH-004 — Email verification & account recovery

- **Status:** Adopted
- **Group:** Authentication
- **Applies to:** Every app with email-based accounts.
- **Last updated:** 2026-08-26

## Requirement

1. Sign-up **MUST** verify the email address (emailed code or link) before the
   account is treated as confirmed.
2. There **MUST** be a self-service **forgot-password** flow: request a code by
   email, then set a new password (which is subject to [AUTH-001](AUTH-001-password-credentials.md)).
   The routes that flow runs on — `/forgot` and `/reset`, their four states, and
   the entry link that makes it reachable — are specified in
   [PAGE-004](../pages/PAGE-004-password-reset-routes.md).
3. Verification and recovery responses **MUST** be neutral and identical whether
   or not the email is registered (see [SEC-005](../security/SEC-005-account-enumeration.md)).
   "If that email has an account, a code is on its way." — never "no such user".
4. Codes/links **MUST** be single-use and time-limited, and the endpoints that
   send or check them **MUST** be rate-limited (see
   [SEC-002](../security/SEC-002-rate-limiting-and-lockout.md)) to prevent inbox
   flooding and code brute-forcing.
5. A successful password reset **SHOULD** revoke existing sessions
   (see [AUTH-003](AUTH-003-session-token-lifecycle.md)).

## Rationale

Verification keeps typo'd/forged addresses out; recovery is table stakes; neutral
responses stop the recovery flow from becoming an account-enumeration oracle.

## Acceptance criteria

- [ ] An unverified account cannot complete sign-in until it confirms.
- [ ] `/forgot` for an unknown email returns the same page/message as for a known one.
- [ ] Reset/verify codes expire and cannot be reused; resends are throttled.

## Implementation notes

- **OpenCycle / OpenOutdoor:** Cognito email code on sign-up; `/forgot` + `/reset` routes; neutral notice; resend/verify/forgot/reset rate-limited; `preventUserExistenceErrors=ENABLED`.
- **WebhookNotification:** verification + reset implemented in `lib/auth.ts` / api routes.
- **Emergency:** no email/password — N/A.
- **Priorize:** backend flow implemented; **Gap:** the landing login form has no
  "Forgot password?" link, so it is unreachable in practice
  ([PAGE-004](../pages/PAGE-004-password-reset-routes.md) §1).
- Related: [[PAGE-004-password-reset-routes]], [[AUTH-001-password-credentials]].
