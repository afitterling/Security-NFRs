# SEC-009 — Dependency & build supply-chain integrity

- **Status:** Proposed
- **Group:** Security
- **Applies to:** Every app, service, and client that installs third-party code
  or runs a build in CI — npm/pnpm workspaces, Expo/EAS builds, Swift Package
  Manager targets, and the CI workflows that deploy them. Complements
  [SEC-001](SEC-001-secrets-management.md) (secrets), [SEC-007](SEC-007-infra-change-integrity.md)
  (the sources that determine the stack), and [SEC-010](SEC-010-sbom-and-vulnerability-response.md)
  (the inventory of what shipped).
- **Last updated:** 2026-09-20

## Requirement

1. Every project that resolves dependencies **MUST** commit its lockfile
   (`package-lock.json`, `pnpm-lock.yaml`, `Package.resolved`, `Podfile.lock`)
   and every install — local, CI, and deploy — **MUST** be a frozen install
   (`npm ci`, `pnpm install --frozen-lockfile`) that fails when the lockfile and
   manifest disagree. A build **MUST NOT** resolve a floating range at build
   time.
2. Every resolved package **MUST** carry an integrity hash in the lockfile, and
   the resolved registry **MUST** be the pinned one. Dependencies from a git URL
   or a tarball **MUST** name an immutable revision (full commit SHA), never a
   branch or a moving tag.
3. Install-time lifecycle scripts of third-party packages **MUST NOT** run
   implicitly in CI or in any automated build: install with scripts disabled
   (`npm ci --ignore-scripts`) and run the small set that is genuinely needed
   from an explicit, reviewed allowlist. Local developer installs **SHOULD**
   follow the same rule.
4. CI actions and reusable workflows **MUST** be pinned to a full commit SHA,
   not to a tag or branch. Workflow token permissions **MUST** be least-privilege
   (default `permissions: contents: read`, widened per job). A workflow that runs
   on untrusted input (`pull_request_target`, issue/comment triggers) **MUST NOT**
   check out and execute the untrusted ref.
5. Deploy credentials **MUST** be short-lived and federated (OIDC role
   assumption) rather than long-lived static access keys stored in the CI
   provider, wherever the target platform supports it.
6. Adding a **new direct dependency MUST** be a reviewed decision recorded in the
   PR: what it is for, what it pulls in transitively, its license, and whether it
   is maintained. A dependency-update bot's PR **MUST NOT** be auto-merged
   without a human reading the lockfile diff.
7. Prebuilt binaries, model files, fonts, and other vendored non-source artifacts
   **MUST** be recorded with their source URL and a content hash, and the hash
   **MUST** be verified at fetch time. They **MUST NOT** be pulled from a mutable
   URL at build or run time.
8. Releases **SHOULD** be traceable from the shipped artifact back to a reviewed
   commit: a signed tag over the commit at minimum, and an attestation binding
   commit → artifact digest where the platform supports it (SLSA/cosign, GitHub
   artifact attestations, App Store build records).

## Rationale

Almost nothing we ship is code we wrote. The realistic compromise path is not an
attacker finding a flaw in our handler — it is a transitive package gaining a new
maintainer, a `postinstall` script reading `~/.aws/credentials` on a CI runner, or
a mutable `@v3` action tag being repointed. Frozen installs plus integrity hashes
make the set of bytes installed today the same set that was reviewed; disabling
lifecycle scripts removes the single most used execution primitive in npm
compromises; SHA-pinned actions and short-lived OIDC credentials remove the two
things such an execution primitive is most often used to steal. Requiring a human
to read a lockfile diff is the control that the others depend on — every
automated check passes on a signed, hashed, frozen malicious dependency.

## Acceptance criteria

- [ ] A clean checkout installs with a frozen install and no network resolution
      of ranges; a hand-edited manifest without a lockfile update fails CI.
- [ ] `git grep` over lockfiles finds no dependency resolved to a branch, a
      moving tag, or a registry other than the pinned one.
- [ ] CI installs with scripts disabled; the allowlist of packages whose scripts
      do run is a committed, reviewable list.
- [ ] Every `uses:` in every workflow is pinned to a 40-character SHA.
- [ ] No long-lived cloud access key exists in CI secrets for any stage that
      supports OIDC.
- [ ] The CI role for a PR build cannot write to production.
- [ ] A vendored binary artifact has a recorded hash, and tampering with it fails
      the build.

## Implementation notes

- **Lockfile scope vs. SEC-007:** [SEC-007](SEC-007-infra-change-integrity.md)
  already folds `package-lock.json` into the per-stage stack fingerprint, so an
  unreviewed lockfile change is a build failure on stacks that adopt it. SEC-009
  is the wider rule — it covers clients and repos with no IaC stack, and covers
  *how* the install runs, not only whether the file changed.
- **npm:** `npm ci --ignore-scripts` in CI; the few packages that need a build
  step (native modules, `sharp`-style postinstalls) get an explicit
  `npm rebuild <pkg>` line rather than a blanket re-enable.
- **Expo / EAS:** the lockfile is the input to the managed build; pin the Expo
  SDK and `expo-*` packages exactly, and treat an EAS build profile change like
  an infra change (reviewed, not ambient).
- **Swift / SwiftPM:** `Package.resolved` is the lockfile and **MUST** be
  committed — Shopping List and the Mac Catalyst targets included. SwiftPM has no
  install-time script execution, so clause 3 is inert there; clauses 1, 2, 6 and
  8 still apply.
- **GitHub Actions:** SHA-pinning is compatible with Dependabot — it rewrites the
  SHA and keeps the human-readable tag as a trailing comment.
- **Open gap:** attestation (clause 8) is `SHOULD` because the Expo/EAS and App
  Store paths do not yet give us a verifiable commit → artifact binding we
  control end to end. Signed tags are the interim floor.
