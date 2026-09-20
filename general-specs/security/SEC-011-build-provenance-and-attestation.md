# SEC-011 — Build provenance & artifact attestation

- **Status:** Proposed
- **Group:** Security
- **Applies to:** Every artifact that reaches a machine we do not build on — a
  deployed Lambda/function bundle, a static asset set on the CDN, a container
  image, an EAS/Xcode client build, a published npm package. Completes the
  supply-chain triple with [SEC-009](SEC-009-dependency-and-build-supply-chain.md)
  (what got in) and [SEC-010](SEC-010-sbom-and-vulnerability-response.md) (what
  shipped); the reviewed-source side is [SEC-007](SEC-007-infra-change-integrity.md).
- **Last updated:** 2026-09-20

## Requirement

1. Every released artifact **MUST** be identified by a **content digest**
   (SHA-256 of the exact bytes), and that digest — not a tag, a branch, a version
   string, or a filename — **MUST** be what provenance is recorded against and
   what deployment refers to.
2. Every released artifact **MUST** carry a **provenance attestation** that binds,
   at minimum: the artifact digest · the source repository and the full commit
   SHA · the build workflow/entry point that produced it · the builder identity
   (which CI system, which runner type) · the build timestamp. An in-toto/SLSA
   provenance predicate is the expected shape.
3. Attestations **MUST** be produced **by the build platform, not by the build
   script** — a step the workflow under attestation can freely author is a claim,
   not provenance. GitHub's `attest-build-provenance` (OIDC-backed) and equivalent
   platform-issued attestations satisfy this; an `echo`-ed JSON file does not.
4. Signing **MUST** use keyless/short-lived OIDC identities (Sigstore/Fulcio) or
   a KMS-held key that CI can use but cannot export. A long-lived private signing
   key stored in a CI secret **MUST NOT** be used.
5. Deployment **MUST** verify before it trusts: the deploy step **MUST** check
   that the artifact digest it is about to promote has a valid attestation from
   the **expected repository, expected workflow, and expected signer identity**,
   and **MUST** fail closed on a missing, unverifiable, or mismatched
   attestation. Producing attestations that nothing verifies is theater.
6. Verification policy **MUST** pin the identity, not just the signature: "signed
   by someone" is not an assertion. A valid attestation from a different repo,
   a fork, or a workflow file other than the expected one **MUST** fail.
7. **Promotion between stages MUST move the artifact, not rebuild it** — the
   digest verified in dev **MUST** be the digest that reaches production. A stage
   that rebuilds from source is producing a different artifact, which no earlier
   verification covers.
8. Provenance records **MUST** be retained and queryable for at least as long as
   the artifact can still be running — the same floor as the SBOM in
   [SEC-010](SEC-010-sbom-and-vulnerability-response.md) clause 3 — and the SBOM
   for an artifact **SHOULD** be attached as an attestation against the same
   digest, so inventory and provenance answer to one identifier.
9. Builds **SHOULD** be reproducible, or failing that, **hermetic**: pinned
   toolchain versions, no network fetches during the build beyond the frozen
   dependency install, no dependence on the runner's ambient state. Where
   reproducibility is achievable, an independent rebuild producing the same
   digest **SHOULD** be checked periodically rather than assumed.
10. Where the distribution channel owns the final signing step and we cannot
    attest the shipped bytes (App Store / TestFlight re-signing, EAS-managed
    builds), the provenance chain **MUST** be recorded up to the last artifact we
    control, and the gap **MUST** be documented in that app's notes rather than
    left implied.

## Rationale

SEC-009 makes the inputs reviewed and SEC-010 makes the contents known, but both
describe a build that happened somewhere we have to take on faith. Provenance is
the link that makes the rest checkable: without it, "this Lambda was built from
commit abc123" is an assumption based on the deploy log, and a compromised runner,
a rerun of an old workflow, or a hand-uploaded bundle produces the same log line.

The clause that does the actual work is verification at deploy time. Producing
attestations is cheap and increasingly automatic; the failure mode in practice is
a pipeline that emits beautiful signed provenance which no gate ever reads, so an
unsigned artifact deploys exactly as smoothly as a signed one. Pinning the
identity matters for the same reason — Sigstore will happily verify a genuine
signature from a fork of our repo built by someone else's workflow.

Rebuild-on-promote is the quiet hole: it defeats every verification performed
earlier in the pipeline while looking like ordinary practice, because the thing
promoted to production is then an artifact no one ever verified.

## Acceptance criteria

- [ ] Every release records an artifact digest, and deploys refer to the digest.
- [ ] `gh attestation verify <artifact> --repo <org>/<repo>` (or the equivalent
      cosign verification) passes for a shipped artifact.
- [ ] Verification with the **wrong** repo, workflow, or signer identity fails.
- [ ] An artifact with no attestation cannot be deployed to any stage — the
      deploy step fails closed rather than warning.
- [ ] A hand-built local bundle uploaded to a stage is rejected.
- [ ] Promotion dev → production moves the identical digest; the production
      deploy log and the dev deploy log name the same artifact.
- [ ] No exportable long-lived signing key exists in CI secrets.
- [ ] The SBOM and the provenance for a release resolve from the same digest.
- [ ] For each client app, the notes state where the attested chain ends and why.

## Implementation notes

- **Relationship to SEC-009 clause 8:** that clause made attestation a `SHOULD`
  and named signed tags as the interim floor. SEC-011 is the full requirement it
  pointed at; SEC-009 clause 8 stays the minimum for repos that have not adopted
  this yet.
- **What SEC-007 does and does not cover:** the stack fingerprint proves the
  *sources* that determine a stack are the reviewed ones, recomputable offline
  from a checkout. It says nothing about the machine that turned those sources
  into a bundle. Fingerprint + provenance together give source → build → artifact;
  either alone leaves a link unverified.
- **GitHub Actions:** `actions/attest-build-provenance` issues a SLSA v1
  provenance predicate signed via Sigstore with the workflow's OIDC identity,
  logged to Rekor; verification is `gh attestation verify` or `cosign
  verify-attestation` with `--certificate-identity-regexp` /
  `--certificate-oidc-issuer` pinned to our org and workflow path. Requires
  `permissions: id-token: write, attestations: write` on that job only — which is
  exactly the least-privilege shape SEC-009 clause 4 asks for.
- **SST / Lambda:** SST deploys from a local build by default, so the natural
  adoption path is to attest the built bundle in CI, and have the deploy job
  verify the digest before `sst deploy`. Clause 7 (promote, don't rebuild) is the
  harder change here and is worth treating as its own piece of work.
- **Static assets:** a CDN asset set is many files; attest the manifest/archive
  digest rather than each file, and make the manifest the deployed unit.
- **Swift / App Store:** Apple re-signs on distribution, so clause 10 applies —
  the attested chain ends at the archive we upload. Record the archive digest and
  the commit, and note the handoff explicitly; App Store Connect's build record
  is the far side of the gap and is not ours to attest.
- **Expo / EAS:** managed builds happen on Expo's infrastructure. Either attest
  the source bundle handed to EAS and accept the gap, or move to a self-hosted
  build to close it — a decision per app, recorded per app.
