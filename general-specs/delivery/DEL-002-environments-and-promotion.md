# DEL-002 — Environments & promotion

- **Status:** Adopted
- **Group:** Delivery
- **Applies to:** All deployed services.
- **Last updated:** 2026-06-16

## Requirement

1. Each app **MUST** have at least a **dev** and a **prod** stage with isolated
   resources (separate tables, pools, secrets). Stages **MUST NOT** share state.
2. Every stage **MUST** have its own secrets (signing keys, session secrets) —
   distinct values, set via the secret store or stage `.env` (see
   [SEC-001](../security/SEC-001-secrets-management.md)). No secret is reused
   across stages or apps.
3. Changes **MUST** be promoted **dev → verify → prod**. Production **MUST NOT**
   be hand-edited. Hot paths (auth, ingest) especially follow this order.
4. The protected production stage name **MUST** be explicit and matched exactly
   in `protect`/`removal` config; a differently-named stage (e.g. `prod` vs
   `production`) **MUST NOT** accidentally bypass protection — confirm the target
   before deploying.
5. A deploy to a user-facing/prod environment is a high-impact action and
   **MUST** be confirmed against the intended stage before running.
6. After deploy, a smoke check **SHOULD** confirm the public surface is healthy
   (key routes return expected status; the function didn't crash on cold start).

## Rationale

Isolated stages + per-stage secrets prevent a dev mistake or leak from touching
production. Explicit, confirmed promotion avoids the classic "wrong stage" deploy.

## Acceptance criteria

- [ ] dev and prod use separate tables/pools/secrets.
- [ ] Each stage's signing/session secret is unique.
- [ ] No process edits production directly; promotion is dev → verify → prod.
- [ ] Post-deploy smoke check passes (e.g. `/login` 200, `/health` ok, bad redirect 400).

## Implementation notes

- **Emergency:** deployed to `dev` and `prod` with distinct `SessionSecret` per stage; protected stage is `production` (untouched).
- **OpenOutdoor:** ported security work deployed to `dev`; per-stage `CAPTCHA_SECRET`; `production` protected.
- **WebhookNotification:** dev + production custom domains; lesson: split dev→verify→prod for ingest hot-path edits.
- **OpenCycle:** code complete, awaiting deploy + dev login test of SRP / exchange / forgot-password.
