# Edge cases & checklist

Cross-cutting traps for the patterns in this folder, plus a consolidated
go-live list.

## Disposable accounts

- **Cleanup gate is the only guard.** `/tests/cleanup` is unauthenticated — its
  `endsWith("@selftest.invalid")` check is *all* that stops it deleting real
  accounts. Normalize (trim + lowercase) before the check, and never widen the
  suffix to a substring (`includes`) — `me@selftest.invalid.evil.com` must fail,
  and `endsWith` handles that.
- **Use a truly unroutable namespace.** `.invalid` (RFC 6761) and
  `example.com`/`.test` are reserved; never use a domain you don't own or a real
  one — a typo could mail a stranger or collide with a live user.
- **Seed must satisfy the real password policy.** The generated password has to
  pass the same validation as a human signup (`Aa1!` + entropy in the
  reference), or login legitimately fails and the "test" is testing nothing.
- **Seed is a user-creation endpoint.** Rate-limit it per IP; without the cap
  it's a table-flooding / cost vector.
- **Prod self-tests touch prod data.** Each run on prod creates and deletes a
  real (inert) row, briefly counts toward user totals, and trips signup
  rate-limit buckets. Decide deliberately: gate `/tests*` to non-prod, or accept
  it and keep the namespace + cleanup tight. If gating, fail closed (404) in
  prod rather than leaving the endpoint live but undocumented.

## E2E

- **Don't assert on absence of error.** Assert the signed-in panel / specific
  error text — a no-op sign-in passes a "no error appeared" check.
- **Live targets flake.** Keep `retries: 1` and trace-on-retry; without retries
  a transient network blip reads as a regression and blocks deploys.
- **Leaked accounts on a crashed run.** If the process dies before `finally`,
  the account lingers (harmless, but accumulates). A periodic sweep of the
  disposable namespace is a cheap safety net.
- **Two origins.** `BASE_URL` (UI) and `AUTH_URL` (API) can differ; defaulting
  both to the same stage avoids a cross-stage test that silently passes.

## Cross-domain auth & deep links

- **Forgetting `x-forwarded-for`** through the proxy collapses every visitor to
  one egress IP — per-user rate-limiting then locks out everyone at once.
- **`fetch` auto-follows redirects.** Without `redirect: "manual"`, a 3xx
  "back to the app" deep link is swallowed by the proxy instead of reaching the
  browser. Relay status + `location`.
- **Relative vs absolute form actions.** A proxied page that posts to an
  absolute function URL escapes the proxy on submit (and may CORS-fail or leak
  the URL). Keep actions relative.
- **Redirect change needs an app rebuild.** `AUTH_REDIRECT_URI` is compiled into
  the native build; a backend deploy alone won't change where sign-in lands.
- **Don't drop legacy schemes too early.** Removing an old scheme from
  `safeRedirect` strands users on already-installed builds. Retire it only after
  those builds age out.
- **`safeRedirect` is an open-redirect boundary.** Allow-list exact scheme
  prefixes; never reflect an arbitrary `redirect_uri` back into a deep link.

## Web write-back

- **Soft-delete, not omission** — omitting a row from a replace-style PUT lets
  other devices resurrect it. Tombstone instead.
- **Spread to preserve unknown fields** — a web client that knows a subset of
  the model must not reconstruct records, or it silently drops the app's
  reminders / links / order on every edit.
- **Read-modify-write race** — concurrent full-list PUTs can lose an update;
  fine for a single-user list, but a per-record API removes both the race and
  the extra GET.
- **Bump `updatedAt`** or the mutation loses last-write-wins to a stale device.

## Go-live checklist

- [ ] `/tests*` either gated to non-prod or knowingly safe on prod.
- [ ] Cleanup endpoint namespace-gated, normalized, `endsWith`.
- [ ] Seed rate-limited; password meets policy.
- [ ] E2E runs against each stage by env var, with positive + negative specs.
- [ ] Auth pages served under the product domain; `x-forwarded-for` forwarded.
- [ ] Per-env deep-link scheme; `safeRedirect` allow-list current.
- [ ] Web deletes are tombstones; edits preserve unknown fields and bump
      `updatedAt`.
