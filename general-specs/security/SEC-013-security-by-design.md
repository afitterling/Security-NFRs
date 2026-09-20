# SEC-013 — Security & privacy by design (and by default)

- **Status:** Proposed
- **Group:** Security
- **Applies to:** Every feature, service, endpoint, client surface, and infra
  change — at **design time**, before implementation. This is the process spec
  the rest of the group depends on: SEC-001…012 each defend one thing, and this
  one decides that they are considered while the design is still cheap to
  change. Legal hook: GDPR Art. 25, *data protection by design and by default*
  ([PRIV-002](../privacy/PRIV-002-gdpr-dsgvo-user-rights.md)).
- **Last updated:** 2026-09-20

## Requirement

### Design time

1. Any change that crosses a **trust boundary** — accepts external input, stores
   or moves personal data, grants access, takes money, or alters infrastructure
   — **MUST** have a written threat sketch **before** implementation, naming:
   the **assets**, the **actors** (including the legitimate user acting against
   us), the **entry points**, what an attacker would try, and what stops them.
2. The sketch **MUST** be sized to the change: a few lines in the PR description
   or the feature spec for a small change, a section for a new surface. A
   ceremony nobody completes protects nothing — the failure mode to avoid is a
   template, not a short answer.
3. A design **MUST** name the **spec IDs it satisfies** and, explicitly, the ones
   it knowingly does not yet meet. A gap is something we **recorded**, not
   something a later audit **discovered**.
4. Changes touching authentication, payment, personal data, or infrastructure
   **MUST** get a second reader **at design time**, not only at PR time. A design
   flaw found at review costs an edit; found at PR it costs the branch; found in
   production it costs an incident and a migration.
5. A **new trust boundary re-opens the sketch**: a new third-party integration, a
   new class of stored data, a new client platform, a new authentication path, or
   a first external consumer of an internal API. The previous sketch's conclusion
   does not carry over to a boundary it never considered.
6. Security-relevant design decisions **MUST** be recorded with their **reason**,
   next to the thing they constrain. An unexplained constraint is removed by the
   next person in good faith — the rationale is the control, not the rule.

### By default

7. The **default MUST be the safe option**: deny by default, sharing off,
   collection off, the most restrictive permission, the shortest retention
   ([PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md)). A user
   who never opens settings **MUST** already be in the safest configuration this
   product supports.
8. Every authorization, verification, signature, and rate-limit check **MUST
   fail closed**. An error, a timeout, or an unavailable dependency resolves to
   **deny**, never to allow. `catch { return true }` is the single highest-yield
   bug class in this category, and a `MUST NOT`.
9. Every identity — function role, API token, OAuth scope, app entitlement, CI
   credential — **MUST** be scoped to the minimum that makes the feature work,
   granted per purpose rather than per convenience, and **MUST NOT** be widened
   to unblock a deploy without the change being reviewed as a security change.
10. **Surface is a liability**: an endpoint, field, parameter, flag, or debug
    affordance **MUST NOT** ship because it might be useful later. Every publicly
    reachable route **MUST** be enumerable and owned by a spec.

### Structural properties

11. The **client is never a trust boundary** — not the web app, not the native
    app, not a partner's server. Every input crossing into our systems **MUST**
    be validated server-side against an explicit schema/allowlist, and every
    per-resource authorization **MUST** be decided from the authenticated
    identity ([SEC-004](SEC-004-transport-and-at-rest.md) clause 6,
    [SEC-012](SEC-012-apple-client-local-attack-hardening.md) clause 23).
12. No single control **MUST** be the only thing protecting a critical asset.
    Where one control is all there is, that fact **MUST** be written down, so the
    thinness is a decision rather than an assumption.
13. Designs **MUST** prefer a **contained blast radius** over a perimeter:
    per-stage and per-app trust domains with distinct keys
    ([SEC-001](SEC-001-secrets-management.md) clause 3), data partitioned by
    user, no shared secret spanning two apps, and no credential that unlocks
    every stage at once.
14. A design **MUST** state what happens **after** a compromise it cannot
    prevent: what can be rotated, what can be revoked, what can be reconstructed,
    and what is simply lost. A key that cannot be rotated and a log that cannot
    answer "what did they reach" are design defects, not operational ones.
15. Specs and acceptance criteria **MUST** state **abuse cases** — what must
    *not* be possible — alongside the happy path, and the negative **SHOULD** be
    asserted by a test. "Signs in with the right password" is not the interesting
    half of the requirement.

## Rationale

Nearly every spec in this group exists because something was built first and
secured afterwards. That order is the expensive one: a missing check is a patch,
but a design that puts the authorization decision on the client, shares one key
across two apps, or stores a credential it never needed to hold is a migration,
and migrations are what actually get deferred.

The point of this spec is not to add a gate. It is that the decisions determining
whether the other twelve specs are even *achievable* are made in the hour before
anyone writes code — where the data lives, who holds the key, which side decides
access, what the default is. After that hour, those answers are load-bearing, and
changing them costs more than the feature did.

Two clauses carry most of the value. **Fail closed** (clause 8), because the
realistic breach is not a defeated control but a control that errored and
returned "allow" — the code path nobody tested, since it only runs when something
else is already broken. And **safe defaults** (clause 7), because the
configuration almost every user runs is the one we shipped; a setting that must
be found and changed protects the small fraction who go looking, which is not the
population that needs protecting.

Clause 6 is the one that keeps the rest alive. A constraint with no recorded
reason reads as an obstacle, and the next person removes it — correctly, on the
information they have.

## Acceptance criteria

- [ ] A change crossing a trust boundary has a threat sketch in the PR or spec,
      naming assets, actors, entry points, and mitigations.
- [ ] The design names the spec IDs it satisfies and the gaps it leaves open.
- [ ] Auth/payment/PII/infra changes show a design-time second reader.
- [ ] Defaults are checked directly: a brand-new account, untouched, is in the
      most restrictive configuration the product offers.
- [ ] Every authorization and verification path has a test that makes the
      dependency fail and asserts **deny**.
- [ ] `git grep` finds no `catch`/`rescue` around a security check that returns
      a permissive value.
- [ ] Each function role, token, and entitlement can be justified clause by
      clause; none was widened to unblock a deploy without review.
- [ ] Every publicly reachable route is enumerated and traceable to a spec.
- [ ] No secret, key, or credential is shared between two apps or two stages.
- [ ] Each critical asset's design states what is rotated, revoked, or lost after
      a compromise.
- [ ] Specs state abuse cases, and at least the critical ones have negative
      tests.

## Implementation notes

- **The mechanism is this repo.** The standing rule in the spec root — consult
  the specs whenever planning, designing, scaffolding, implementing, or reviewing
  — is what makes clauses 1 and 3 happen in practice rather than aspirationally.
  Answering "which specs apply here?" at the start of a feature **is** the
  lightweight threat sketch for most changes; the written sketch is for the ones
  where the answer is not obvious.
- **Sizing clause 1:** for a new endpoint, three lines — what it reads, who may
  call it, what a caller could do that we would not want. For a new surface
  (payment, a new client platform, a first external API consumer), a section with
  the boundary drawn explicitly. Anything longer is usually a sign the design
  itself is unclear, which is the useful finding.
- **Fail-closed review (clause 8):** worth a deliberate pass over every
  `try`/`catch` wrapping token verification, signature checks, entitlement
  lookups, and rate limiters. The bug is rarely visible in the happy path and
  never in the tests, because it only executes when a dependency is down.
- **Defaults review (clause 7):** check on a freshly created account, not on a
  developer account that has been toggled for months — the two have different
  configurations and only one of them ships.
- **Relationship to the rest of the group:** SEC-013 is the design-time entry
  point; [SEC-009](SEC-009-dependency-and-build-supply-chain.md) /
  [SEC-010](SEC-010-sbom-and-vulnerability-response.md) /
  [SEC-011](SEC-011-build-provenance-and-attestation.md) cover the build and what
  it produced; [SEC-012](SEC-012-apple-client-local-attack-hardening.md) covers
  the device it lands on; SEC-001…008 cover the running service.
- **Not in scope:** a formal STRIDE/LINDDUN exercise, a risk register, or a
  compliance artifact. Those are for a size of organization we are not, and the
  version of this spec that demanded them would be ignored — which is worse than
  the version that asks for three honest lines.
