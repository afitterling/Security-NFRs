# Headless E2E login tests (Playwright)

The same disposable-account idea as the [self-test page](self-test-page.md), but
driving the **browser `/login` UI** headlessly so it runs in CI and gates a
deploy. One spec asserts a good password signs in; one asserts a wrong password
is rejected. Both seed from the API and clean up in a `finally`.

Reference impl: `sst/test/login.spec.ts`, `sst/playwright.config.ts`.

## Config — point at a deployed stage via env

The suite has **no app under test of its own**; it targets a deployed stage.
Two URLs, both overridable, defaulting to the dev stage:

```ts
// playwright.config.ts
export default defineConfig({
  testDir: "./test",
  timeout: 30_000,
  fullyParallel: true,
  retries: 1,                                  // flaky-network tolerance for a live target
  reporter: [["list"]],
  use: {
    baseURL: process.env.BASE_URL ?? "https://<dev-landing-domain>",   // where /login lives
    trace: "on-first-retry",
  },
  projects: [{ name: "chromium", use: { ...devices["Desktop Chrome"] } }],
});
```

```ts
// login.spec.ts — the Auth function URL (seed/cleanup + authorize live here)
const AUTH_URL = (process.env.AUTH_URL ?? "https://<dev-auth-lambda-url>").replace(/\/$/, "");
```

Run dev with defaults; run prod with `BASE_URL=… AUTH_URL=… npx playwright test`.
Two URLs because the **browser UI** (`/login`) and the **API** (`/tests/seed`)
can live on different origins (landing domain vs. Auth function URL).

## The happy-path spec

```ts
test("logs in with a confirmed account seeded from the API", async ({ page, request }) => {
  // 1. Seed via the API — Playwright's `request` fixture, not the browser.
  const seedRes = await request.get(`${AUTH_URL}/tests/seed`);
  expect(seedRes.ok(), "seed endpoint should be available").toBeTruthy();
  const { email, password } = await seedRes.json();
  expect(email).toContain("@selftest.invalid");        // guard: never run against a real address

  try {
    await page.goto("/login");
    await page.getByRole("tab", { name: /log in/i }).click();
    await page.locator("#email").fill(email);
    await page.locator("#password").fill(password);
    await page.getByRole("button", { name: /log in & sync/i }).click();
    // Assert the signed-in panel, not just "no error".
    await expect(page.getByText(/signed in as/i)).toBeVisible({ timeout: 15_000 });
    await expect(page.getByText(email)).toBeVisible();
  } finally {
    await request.post(`${AUTH_URL}/tests/cleanup`, { data: { email } });   // always clean up
  }
});
```

The negative spec is the same shape: seed, fill a deliberately wrong password,
and assert the **specific** error (`/wrong email or password/i`) is visible.

## Conventions that make this reliable

- **Seed via `request`, drive via `page`.** Account setup goes through the API
  fixture (fast, deterministic); only the thing under test goes through the UI.
- **Assert the post-state, not the absence of an error.** "Signed in as <email>"
  visible — a green-on-no-error test passes even when sign-in silently no-ops.
- **`finally` cleanup.** A failed assertion still deletes the account; the
  namespace gate on `/tests/cleanup` makes that delete safe.
- **`retries: 1` + `trace: "on-first-retry"`.** Live targets have transient
  network flakes; one retry with a trace captured only on the retry keeps CI
  green without hiding real regressions.
- **Role/label selectors** (`getByRole("tab", …)`, `getByRole("button", …)`)
  over CSS — they double as a light
  [accessibility](../general-specs/accessibility/A11Y-001-baseline.md) check.
- **Negative path included.** Verifying that wrong credentials are *rejected* is
  as important as verifying the happy path — it proves the guard is wired.

## Checklist

- [ ] Target stage chosen by env var; sane non-prod default.
- [ ] Account seeded from the API, asserted to be in the disposable namespace.
- [ ] Both a positive (signs in) and a negative (rejected) spec.
- [ ] Assertions check the resulting signed-in state, not merely no-error.
- [ ] Cleanup in `finally`.
- [ ] `retries` + trace-on-retry for live-target flakiness.
