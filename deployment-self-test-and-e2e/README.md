# Deployment Self-Test, E2E & Per-Stage Auth Verification

Prove that the **deployed** sign-in flow actually works — on each stage, in a
real browser, with real PKCE — using *disposable* accounts that can never
receive mail and are always cleaned up. Plus the two things that make a
multi-stage auth flow correct in the first place: serving the auth pages under
your own domain, and keeping each environment's deep-link redirect from
colliding with the others'.

This is a reusable, parameterized recipe distilled from the last cluster of SST
commits on `threethings` (SST + Hono backend in `sst/src/`, Remix landing in
`sst/landing page/`, Expo app in `iosapp/`). Swap in your own stage URLs, table,
and bundle id to reuse it.

> Status: **proven**. All four pieces are implemented and committed:
> `6b2ea32` (Playwright e2e), `e5fd99b` (`/tests` self-test page + env-specific
> redirect), `1ebef27` (auth pages under the landing domain), `ec615f4` (web
> write-back). The web write-back doc is grouped here because the same commit
> cluster shipped it and it shares the last-write-wins discipline the e2e tests
> exercise.

## Why these belong together

A sign-in flow that works on `localhost` routinely breaks once deployed: the
redirect scheme is wrong for that stage, the auth page lives on a raw Lambda
URL, rate-limiting keys on the wrong IP, or the PKCE exchange silently 400s. The
only reliable check is to **run the real flow against the real stage**. These
docs give you (a) two ways to run it — headless E2E in CI and a one-click
in-stage page — and (b) the two correctness fixes that the runs would otherwise
keep catching.

```
Disposable account  →  real PKCE sign-in  →  assert PASS  →  always clean up
   /tests/seed           /web/authorize        signed-in       /tests/cleanup
 @selftest.invalid       /auth/exchange          panel        (namespace-gated)
```

## Folder index

| Area | Doc |
|---|---|
| In-stage **`/tests`** live PKCE self-test page + seed/cleanup endpoints | [self-test-page.md](self-test-page.md) |
| **Playwright** headless e2e login tests (CI, per-stage) | [e2e-login-tests.md](e2e-login-tests.md) |
| Serve auth pages under **your own domain** + per-env deep-link redirect | [cross-domain-auth-pages.md](cross-domain-auth-pages.md) |
| **Web write-back** to a last-write-wins sync (edit / soft-delete) | [web-write-back-sync.md](web-write-back-sync.md) |
| Edge cases & checklist | [gotchas.md](gotchas.md) |

## The disposable-account convention (used by every test here)

- **Namespace:** `…@selftest.invalid`. `.invalid` is a reserved TLD (RFC 6761) —
  it can never receive real mail, so a leaked/forgotten test account is inert.
- **Seeded already-confirmed:** the seed endpoint sets `confirmedAt` so the
  test account clears the [double opt-in](../double-opt-in-auth/README.md) login
  guard without an email round-trip.
- **Cleanup is namespace-gated:** the delete endpoint refuses any address that
  doesn't end in `@selftest.invalid` — it can *only* delete test accounts.
- **Always clean up:** tests delete in a `finally`/`try` so a failed assertion
  still removes the account. The self-test page deletes at the end of its run.
- **Rate-limited seed:** the seed endpoint is rate-limited per IP so it can't be
  used to flood the user table.

## Decisions (locked)

- **Run against deployed stages, not a local mock.** The whole point is to catch
  stage-specific breakage (redirect scheme, domain, rate-limit key).
- **Two harnesses, one flow.** Playwright drives the *browser* `/login` UI;
  `/tests` drives the *API* (`/web/authorize` → `/auth/exchange`) directly. Keep
  both — they fail for different reasons.
- **Auth pages live under the product domain**, reverse-proxied to the Auth
  function; the raw Lambda Function URL is never linked to a user.
- **Deep-link redirect scheme is per-environment** (`<bundle-id>://auth`) and
  the backend `safeRedirect` allow-list enumerates every accepted scheme.
- **Web mutations are full-list write-backs** (fetch-all → change-one → PUT) and
  deletes are **soft-delete tombstones** so they win last-write-wins sync.

## Reference implementation (in the repo)

- `sst/src/index.ts` — `/tests`, `/tests/seed`, `/tests/cleanup`.
- `sst/src/page.ts` — `renderTestsPage()` (the live PKCE self-test page).
- `sst/test/login.spec.ts`, `sst/playwright.config.ts` — headless e2e.
- `sst/landing page/app/lib/auth-proxy.server.ts`, `…/routes/forgot.tsx`,
  `…/routes/reset.tsx`, `…/routes/login.tsx` (`safeRedirect`) — domain + scheme.
- `iosapp/src/config.ts` — `AUTH_REDIRECT_URI = \`${BUNDLE_ID}://auth\``.
- `sst/landing page/app/routes/app.tsx` (`mutateTask`) — web write-back.
