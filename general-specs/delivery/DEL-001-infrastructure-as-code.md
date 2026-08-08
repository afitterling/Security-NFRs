# DEL-001 — Infrastructure as code

- **Status:** Adopted
- **Group:** Delivery
- **Applies to:** All deployed services.
- **Last updated:** 2026-06-16

## Requirement

1. All cloud infrastructure **MUST** be defined as code (SST/Ion on AWS) and
   committed. No production resource is created by hand in the console.
2. Resource access **MUST** be least-privilege: functions get only the specific
   IAM actions and resources they need (scoped to the table/pool ARN), never `*`
   except where the API genuinely requires it (e.g. SES send).
3. Runtime configuration **MUST** be injected (linked resources / env / secrets),
   never hard-coded; resource names/URLs are discovered at runtime, not pasted.
4. Region and account **MUST** be explicit and consistent (`eu-central-1`,
   account `327261196437`).
5. Stateful production stacks **MUST** be marked `protect` + `removal: retain`
   so they cannot be torn down by an errant deploy.
6. Custom domains use manually-validated ACM certs (us-east-1 for CloudFront) with
   DNS managed in Vercel; the router activates only when both domain and cert exist.

## Rationale

IaC makes environments reproducible and reviewable; least-privilege and protected
prod stacks contain the blast radius of mistakes and compromises.

## Acceptance criteria

- [ ] A stage can be recreated from code alone.
- [ ] Function IAM policies name specific actions + resource ARNs.
- [ ] No hard-coded resource ids/URLs/secrets in code.
- [ ] Production stack has `protect: true` and `removal: retain`.

## Implementation notes

- **All apps:** `sst.config.ts` defines Cognito/Dynamo/Functions/Crons; resources `link`-ed; IAM scoped to pool/table ARNs; `eu-central-1`.
- **OpenCycle / OpenOutdoor:** new auth Dynamo table + scoped Cognito perms (incl. ForgotPassword/GlobalSignOut); client ids discovered via `ListUserPoolClients`.
- **Emergency / WebhookNotification:** `protect`/`retain` on the `production` stage.
