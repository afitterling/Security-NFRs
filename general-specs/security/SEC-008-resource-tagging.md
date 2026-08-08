# SEC-008 — Resource tagging

- **Status:** Adopted
- **Group:** Security
- **Applies to:** All deployed cloud resources.
- **Cross-cutting:** Also a delivery/IaC concern — tags are applied through the
  infra definition. Complements [DEL-001](../delivery/DEL-001-infrastructure-as-code.md)
  (infrastructure as code), [SEC-001](SEC-001-secrets-management.md) (secrets),
  and [SEC-007](SEC-007-infra-change-integrity.md) (infra change integrity).
- **Last updated:** 2026-06-20

## Requirement

1. Every taggable cloud resource **MUST** carry a consistent set of default tags
   applied at the provider level, not per-resource, so coverage is automatic and
   no new resource is left untagged.
2. The default tag set **MUST** include at minimum:
   - `Project` — the app/product identifier (e.g. `ejectify`).
   - `Stage` — the deploy stage (`production`, `dev`, …), resolved at deploy time.
   - `ManagedBy` — the provisioning tool (`sst`).
3. Default tags **MUST** be configured on the AWS provider in `sst.config.ts`
   (`providers.aws.defaultTags`) so they flow to all resources on the default
   provider.
4. Resources created with an **explicit** (non-default) provider — e.g. the
   `us-east-1` provider required for CloudFront-scoped WAF Web ACLs — **MUST**
   replicate the same `defaultTags` on that provider, since default-provider tags
   do not propagate to explicit providers.
5. Tagging **MUST NOT** trigger resource replacement: tags apply in place on the
   next deploy.

## Rationale

Consistent ownership tags are a governance and account-hygiene control: they enable
cost allocation and cost-anomaly detection, blast-radius scoping during an incident,
console/CLI filtering, and clean separation of these resources from others sharing
the AWS account. Applying them as provider defaults guarantees coverage without
per-resource discipline. SST's own `sst:app` / `sst:stage` tags are machine-facing;
the `Project`/`Stage`/`ManagedBy` set is the human- and billing-facing complement.

## Acceptance criteria

- [ ] `providers.aws.defaultTags` sets `Project`, `Stage`, `ManagedBy` in `sst.config.ts`.
- [ ] Any explicit-provider resource (e.g. the us-east-1 WAF) carries the same tags.
- [ ] A deploy/diff that adds tags shows in-place updates only, no replacements.
- [ ] Spot-checking a DynamoDB table and (in prod) the WAF Web ACL shows the full tag set.

## Implementation notes

- **ejectify-landing:** `sst.config.ts` sets `providers.aws.defaultTags.tags =
  { Project: "ejectify", Stage: input.stage, ManagedBy: "sst" }` in `app()`; the
  `WafUsEast1` explicit provider repeats the same tags with `Stage: $app.stage`.
- Verify with `aws dynamodb list-tags-of-resource --resource-arn <table arn>` and
  `aws wafv2 list-tags-for-resource --resource-arn <acl arn> --region us-east-1`.
