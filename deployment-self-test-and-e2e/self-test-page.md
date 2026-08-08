# In-stage live sign-in self-test (`/tests`)

A page hosted **by each stage** that runs the real PKCE sign-in flow against
*that* stage and reports PASS/FAIL per step, using an ephemeral account it seeds
and then deletes. Open `https://<stage-auth-url>/tests` after any deploy for a
one-click "is sign-in alive here?" smoke test — no CI, no local tooling.

Reference impl: `renderTestsPage()` in `sst/src/page.ts`; the endpoints in
`sst/src/index.ts`.

## The three endpoints

```ts
// A page that, in the browser, runs the live flow and renders PASS/FAIL rows.
app.get("/tests", (c) => c.html(renderTestsPage()));

// Seed an ephemeral, already-CONFIRMED test account. Rate-limited per IP.
app.get("/tests/seed", async (c) => {
  if (!(await allow(`tests:ip:${clientIp(c)}`, { limit: 30, windowSec: 600 }, Date.now())))
    return c.json({ error: "Too many requests — wait a few minutes." }, 429);
  const email = `selftest-${randomBytes(8).toString("hex")}@selftest.invalid`;
  const password = `Aa1!${randomBytes(10).toString("hex")}`;           // meets the password policy
  const { salt, hash } = hashPassword(password);
  await createUser({ email, salt, hash, createdAt: Date.now(), confirmedAt: Date.now() });
  return c.json({ email, password });
});

// Delete — namespace-gated so it can ONLY ever remove a test account.
app.post("/tests/cleanup", async (c) => {
  const body = (await c.req.json().catch(() => null)) as { email?: string } | null;
  const email = String(body?.email ?? "").trim().toLowerCase();
  if (!email.endsWith("@selftest.invalid")) return c.json({ error: "forbidden" }, 403);
  await deleteUser(email);
  return c.json({ ok: true });
});
```

## What the page actually verifies

The page is plain inline JS (no build step, no deps) so it runs anywhere the
stage is reachable. It asserts each hop of the real flow:

1. **Seed** — `GET /tests/seed` returns `{ email, password }`.
2. **Signup is double-opt-in** — `POST /web/authorize {mode:"signup"}` returns
   `200 { pending: true }` (account parked, not active — proves the
   [confirmation gate](../double-opt-in-auth/signup-confirmation.md) is on).
3. **Login returns a PKCE code** — generate a 48-byte `code_verifier`, derive
   `code_challenge = base64url(sha256(verifier))` via `crypto.subtle`, then
   `POST /web/authorize {mode:"login", …, code_challenge}` → `200 { code }`.
4. **Exchange returns a token** — `POST /auth/exchange {code, code_verifier}` →
   `200 { token }` (proves PKCE round-trips end to end).
5. **Cleanup** — `POST /tests/cleanup {email}` removes the seeded account.

PASS rows render lime, FAIL rows render vermilion, each with the HTTP status /
error so a failure is self-describing. See [AUTH-002 — no tokens in URLs (PKCE)](
../general-specs/authentication/AUTH-002-no-tokens-in-urls.md).

## Why each rule matters

- **`.invalid` namespace** — RFC 6761 reserved TLD; the address can't receive
  mail, so an account that escapes cleanup is harmless and obviously synthetic.
- **Seeded `confirmedAt`** — bypasses the email round-trip so the test can log in
  immediately, while still exercising the *login* confirmation guard.
- **Cleanup gated on the suffix** — the delete endpoint is unauthenticated, so
  the namespace check is the only thing standing between it and real accounts.
  It must reject anything not ending in `@selftest.invalid`. **No exceptions.**
- **Rate-limited seed** — an open "create a user" endpoint is a table-flooding
  vector; bucket it per IP (here 30 / 10 min). Reuse your
  [rate-limit](../general-specs/security/SEC-002-rate-limiting-and-lockout.md)
  primitive.
- **Per stage** — each stage hosts its own `/tests`, so the page proves *that*
  environment's redirect scheme, domain, and config — not a shared mock.

## Reuse checklist

- [ ] Seed creates a confirmed account in a reserved/disposable namespace.
- [ ] Seed is rate-limited per IP.
- [ ] Cleanup refuses any address outside the disposable namespace.
- [ ] The page exercises the *real* endpoints (signup→pending, login→code,
      exchange→token), not a stubbed happy path.
- [ ] Each step renders its own PASS/FAIL + status; failures are self-describing.
- [ ] Cleanup runs even when an earlier step fails.
- [ ] Consider gating `/tests*` to non-prod, or accept that prod self-tests
      create+delete a real (inert) row each run — see [gotchas](gotchas.md).
