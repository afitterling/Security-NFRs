# SEC-001 — Secrets management

- **Status:** Adopted
- **Group:** Security
- **Applies to:** All apps and all services.
- **Last updated:** 2026-06-16

## Requirement

1. Secrets (signing keys, API tokens, DB credentials, IdP private keys) **MUST
   NOT** be committed to source control. They live in `.env*` files (git-ignored)
   and/or the platform secret store (SST Secrets), injected at deploy/runtime.
2. Code **MUST NOT** contain an insecure fallback for a security-critical secret
   (e.g. `process.env.X || "dev-insecure"`). A missing/weak secret **MUST** fail
   loudly (throw) rather than silently sign with a guessable key.
3. HMAC/signing keys **MUST** be ≥ 16 bytes of entropy and **MUST** be **distinct
   per app and per stage** — never shared across apps (separate trust domains).
4. Distinct concerns signed with the same key **MUST** be domain-separated
   (namespace prefix per use: CAPTCHA vs OTP vs CSRF vs OAuth-state).
5. Secrets **MUST NOT** be logged, returned in responses, or placed in URLs.
6. Third-party tokens granting broad access **SHOULD** be scoped to the minimum
   needed and, where the surface allows, fronted by a proxy that enforces the
   scope server-side.

## Rationale

A predictable signing key lets anyone forge CAPTCHA/CSRF/session tokens; a shared
key means one app's leak compromises the others. Failing loudly turns a silent
catastrophic default into an obvious deploy-time error.

## Acceptance criteria

- [ ] `git grep` finds no real secret values or `|| "dev-…"` security fallbacks.
- [ ] Booting a function without its signing secret throws, rather than serving.
- [ ] Each app/stage has its own key; rotating one doesn't touch another.
- [ ] CAPTCHA, OTP, CSRF, and OAuth-state signatures cannot be cross-substituted.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `secret.ts` reads `CAPTCHA_SECRET`, throws if absent/short; CAPTCHA/OTP/CSRF/OAuth-state are namespaced; per-app, per-stage `.env` values + `CaptchaSecret` SST Secret.
- **Emergency:** `GA_SESSION_SECRET` insecure fallback removed (throws on missing/weak); per-stage `SessionSecret` SST Secret.
- **ClickUp API:** ClickUp token in git-ignored `.env.local`; proxy restricts access to a whitelisted folder (server-enforced scope).
- **WebhookNotification:** `AUTH_SECRET` SST Secret (strong value required in prod).
