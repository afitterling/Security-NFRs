# REL-005 — Diagnostics surface for support

- **Status:** Proposed
- **Group:** Reliability
- **Applies to:** All client apps.
- **Last updated:** 2026-06-18

## Requirement

1. An app **SHOULD** expose a diagnostics / details surface (in settings or
   about) that shows the values support needs to triage an issue: app **version +
   build**, app/bundle id, **environment**, backend host, data-model/schema
   version, last migration, and last sync.
2. These values **MUST** be read at runtime from the platform (e.g. the native
   build version, the runtime app id), **not** hardcoded, so they reflect the
   actually-installed binary rather than a source constant that may have drifted.
3. The diagnostics **MUST** be provided by a single shared source so the same
   rows feed both this surface and the crash report
   ([REL-004](REL-004-client-crash-handling.md)).
4. The surface **MUST NOT** show secrets or over-share: no tokens, no full signed
   URLs, no raw emails — only non-sensitive support identifiers (e.g. the backend
   **subdomain**, not the full Function URL).

## Rationale

Most support tickets are unanswerable without "what build are you on, against
which backend." Surfacing those truthfully — read from the binary, not from a
constant — collapses a back-and-forth into a glance, and sharing one provider
with the crash screen guarantees they never disagree.

## Acceptance criteria

- [ ] Settings/about shows version+build, bundle id, environment, backend, schema version, last migration, last sync.
- [ ] Version/build/bundle id are read at runtime from the platform, not hardcoded.
- [ ] The crash report and this surface render identical diagnostics from one source.
- [ ] No token, full signed URL, or raw PII appears.

## Implementation notes

- **Nilo:** Settings › DETAILS renders `getDiagnostics()` (`src/diagnostics.ts`): App version `1.2.2 (2)` (build via `expo-application` `nativeBuildVersion`), Bundle ID (`applicationId`), Data model `v1`, Migrations, Last migration (or "never"), Sync backend shown as **subdomain only**. Same provider feeds the crash screen ([REL-004](REL-004-client-crash-handling.md)). Commits `d1d8621`, `800c651`, `edbfeae`.
