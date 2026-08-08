# SEC-007 — Infrastructure change integrity (stack fingerprint)

- **Status:** Adopted
- **Group:** Security
- **Applies to:** Any IaC-defined deployment (SST / pulumi / Terraform) where the
  set of sources that determine the deployed stack should not change without
  review. Complements [SEC-001](SEC-001-secrets-management.md) (secrets) and the
  deploy-identity controls.
- **Last updated:** 2026-06-20

## Requirement

1. The git-tracked sources that **determine the stack** (infra definition,
   build/dependency lockfiles, and the code bundled into the deployed
   functions/assets) **MUST** be reducible to a single deterministic
   **fingerprint** that any checkout can recompute offline.
2. The fingerprint **MUST** be computed over an **explicit allowlist** of paths
   (not "the whole repo"), so reviewers can see precisely what is in scope, and
   **MUST** be order-independent and canonical (sorted inputs, content-addressed
   leaves).
3. The fingerprint **MUST** be **per-stage** (domain-separated by stage name) so
   one stage's pinned value can't be replayed against another, and so each
   stage's expectation is independently reviewable.
4. Stage-specific secret/config inputs that CI cannot reproduce (e.g. gitignored
   `.env.<stage>`) **MUST NOT** be folded into the fingerprint — it stays
   reproducible from a clean checkout. Their integrity is covered by
   [SEC-001](SEC-001-secrets-management.md).
5. Expected fingerprints **MUST** be committed (pinned) and a **verify** step
   **MUST** run in CI **for every stage**; a mismatch **MUST fail the build** and
   **MUST** name the files that drifted.
6. Re-pinning **MUST** be an explicit, reviewable action (a deliberate command +
   committed diff), never automatic — the pin is the human-reviewed root of
   trust.
7. The construction **SHOULD** be second-preimage resistant (RFC-6962-style leaf
   vs. node domain separation) so a leaf can't be reinterpreted as an inner node.

## Rationale

CI lacks a machine-readable resolved infra plan for some tools (`sst diff` has no
`--json`), so the reproducible control is to fingerprint the **sources that
determine** the stack. A per-stage, committed, CI-verified fingerprint makes any
unreviewed change to infra-determining inputs a hard build failure — closing the
gap between "code was reviewed" and "the exact bytes that build the stack are the
reviewed ones." Keeping secret inputs out keeps the fingerprint reproducible;
explicit re-pinning keeps the pin a human decision, not a rubber stamp.

## Acceptance criteria

- [ ] `fingerprint:verify --stage <s>` recomputes the root offline and passes
      against the committed pin for every stage.
- [ ] Changing any in-scope file without re-pinning fails verify and lists the
      added / changed / removed files.
- [ ] dev and production produce **different** roots for identical sources
      (stage domain separation).
- [ ] A CI job runs verify per stage (matrix) and blocks merge on mismatch.
- [ ] Re-pinning is a separate, committed change, not a CI side effect.

## Implementation notes

- **PingTray (`sp33c-landing`):** `sst/scripts/stack-fingerprint.mjs` builds an
  RFC-6962 Merkle tree over an allowlist (`sst.config.ts`, `package.json`,
  `package-lock.json`, `tsconfig.json`, `vite.config.ts`, `app/`, `public/`)
  using `git ls-files`; root is folded with the stage name for domain
  separation. Pins committed at `sst/fingerprints/<stage>.json`; npm scripts
  `fingerprint` / `fingerprint:write` / `fingerprint:verify`. CI:
  `.github/workflows/stack-verify.yml` runs `--verify` in a `[dev, production]`
  matrix on PRs and pushes touching `sst/**`. No dependencies — node builtins +
  git only.
- **Relationship to signing:** a signed git tag over the commit covers source
  provenance; this NFR adds a scoped, per-stage, normalized pin on top. For full
  artifact provenance, pair with cosign/SLSA attestation binding commit → bundle
  hash → fingerprint (out of scope here).
