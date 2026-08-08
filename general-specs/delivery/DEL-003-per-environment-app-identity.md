# DEL-003 — Per-environment app identity & builds

- **Status:** Proposed
- **Group:** Delivery & infrastructure
- **Applies to:** Mobile/desktop apps shipped in more than one stage (dev/prod).
- **Last updated:** 2026-06-18

## Requirement

1. Each non-production stage **MUST** have a distinct app identity — a bundle/app
   id suffix and a distinct display name — so a dev or staging build installs
   **alongside** the store app without overwriting it.
2. The active environment **MUST** be chosen by a single build-time switch (an
   env var or build profile) that selects the matching config bundle (`.env`,
   backend URL, API keys). **Production MUST be the safe default** when the
   switch is unset.
3. Only the **production** identity is shippable to the store. Non-prod
   identities are for device installs / internal TestFlight only and **MUST NOT**
   be submitted for public release.
4. The selected environment **MUST** be visible at runtime in the diagnostics
   surface ([REL-005](../reliability/REL-005-diagnostics-surface.md)).
5. Store-bound resources (IAP/entitlement products, push certificates) are tied
   to the production identity; non-prod builds **MUST** degrade gracefully
   (e.g. purchases disabled) rather than crash when those resources are absent.

## Rationale

A separate dev identity lets a tester keep the real store app and a debug build
on the same device, pointed at separate backends, with zero risk of one
clobbering the other or a dev build leaking to production. A single switch with a
prod-default makes the safe path the easy one.

## Acceptance criteria

- [ ] Dev and prod builds install side by side (distinct bundle id + name).
- [ ] One env switch selects backend/keys; unset ⇒ production.
- [ ] Only the prod identity is ever submitted to the store.
- [ ] The running environment is shown in diagnostics.
- [ ] A non-prod build with no store products runs without purchases instead of crashing.

## Implementation notes

- **Nilo:** `app.config.js` reads `APP_ENV` → prod = `tech.sp33c.nilo` / "Nilo", dev = `tech.sp33c.nilo.dev` / "Nilo Dev"; `.env.prod` / `.env.dev` select backend (`tick` vs `tick-dev`) and the RevenueCat key; `APP_ENV` defaults to `prod`. The env is shown in Settings ([REL-005](../reliability/REL-005-diagnostics-surface.md)); RevenueCat is a no-op when no key is set, so dev builds run without purchases. Commits `e450047`, `95cc212`, `64b93f5`. See [DEL-002](DEL-002-environments-and-promotion.md).
