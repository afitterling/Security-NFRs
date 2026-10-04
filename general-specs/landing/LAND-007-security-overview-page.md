# LAND-007 — Public security overview (`/security`)

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Every app that handles accounts, credentials or user content
  behind a public endpoint, which covers every app with a backend. Optional for
  account-free, local-only apps.
- **Last updated:** 2026-10-04

## Requirement

### Route

1. A security overview **MUST** be served at `/security` on the production host:
   HTTPS, `200`, signed-out, no consent gate, indexable, and listed in the
   sitemap ([SEO-002](../seo/SEO-002-sitemap.md)).
2. It **MUST** be linked from the landing nav
   ([LAND-003](LAND-003-responsive-navigation.md)), from the unified footer
   ([UI-008](../design/UI-008-unified-footer.md)), and from the in-app
   settings/about screen, all through one exported `SECURITY_URL`.
3. The landing page **SHOULD** carry a short **security summary section** (about
   six named controls, one sentence each), ending in a link to the full page
   ([LAND-001](LAND-001-landing-page-anatomy.md) §2).

### Content

4. **Named controls, not adjectives.** Each entry names a concrete mechanism and
   what it prevents: "passwords are hashed with scrypt and a per-password salt",
   not "bank-grade security". Words like "military-grade" or "unhackable" **MUST
   NOT** appear.
5. **Grouped the way an evaluator thinks.** Controls **SHOULD** be grouped under
   these headings, as they apply:
   - accounts and sign-in;
   - credentials and keys;
   - inbound traffic;
   - outbound traffic (callbacks, webhooks we send);
   - the request itself (rate limits, validation, CSRF);
   - your data (storage, encryption, retention, export, deletion, region).
6. **Every claim is true of the shipped code.** Each control **SHOULD** carry the
   internal spec id it implements (`AUTH-003`, `SEC-005`), so a statement maps to
   a change in the code and can be audited. Removing or weakening a control
   **MUST** update the page in the same release
   ([LAND-012](LAND-012-truthful-marketing-claims.md)).
7. **Not an attacker's worksheet.** The page **MUST NOT** publish exact
   thresholds (burst window sizes, per-endpoint ceilings, lockout counts), known
   unfixed weaknesses, or dates that tie a fix to a window of exposure. It
   describes them qualitatively ("a burst guard, a daily quota tied to your plan,
   automatic shutoff of a runaway webhook").
8. **Data rights are linked, not just described.** The data section **MUST**
   link the working export and deletion entry points
   ([PRIV-002](../privacy/PRIV-002-gdpr-dsgvo-user-rights.md),
   [PAGE-001](../pages/PAGE-001-privacy-route.md) §11).

### Vulnerability reporting

9. The page **MUST** end with a **"Reporting a vulnerability"** section. It names
   a working channel (the support route,
   [PAGE-003](../pages/PAGE-003-support-route.md), or a dedicated address), says
   what to include, and asks for coordinated disclosure.
10. The host **SHOULD** serve `/.well-known/security.txt`
    ([RFC 9116](https://www.rfc-editor.org/rfc/rfc9116)) with `Contact:`
    (pointing at the same channel), `Expires:` (at most 1 year out, renewed by a
    release-checklist item), `Policy:` (pointing at `/security`) and
    `Preferred-Languages:`.

### Language

11. The page **MAY** be English-only
    ([LAND-004](LAND-004-language-and-locale-persistence.md) §7). A
    half-translated security page is worse than an untranslated one. If it is
    English-only, it declares `lang="en"`.

## Rationale

Developers and businesses decide whether to send their data to a service partly
by reading how it is protected. A page of named, auditable controls earns that
trust, and a page of adjectives loses it. Tying each claim to a spec id keeps the
page honest over time: when a control changes, the id says which sentence to
revisit. The omissions in §7 are what keep a transparency page from becoming a
map for staying under the radar.

## Acceptance criteria

- [ ] `/security` returns 200 signed-out, is in the sitemap, and is linked from the nav, the footer and the app.
- [ ] Every entry names a mechanism. There is no "bank-grade", "military-grade" or similar adjective.
- [ ] Each control maps to a spec id or to code, and a spot-check of three controls finds each one in the shipped code.
- [ ] The page publishes no numeric rate-limit thresholds, no unfixed weaknesses and no exposure dates.
- [ ] The data section links working export and deletion entry points.
- [ ] A "Reporting a vulnerability" section names a working channel.
- [ ] `/.well-known/security.txt` exists, with `Contact`, `Expires` (in the future), `Policy` and `Preferred-Languages`.

## Implementation notes

- **WebhookPush** is the reference: `routes/security.tsx` has six groups of
  controls with `ref` tags (`AUTH-002`, `AUTH-003`, `SEC-004`, `SEC-005`, …), and
  states the deliberate omissions in its header comment (no thresholds, no
  unfixed weaknesses, no dates). The landing page's `#security` section has six
  named controls and links to `/security`. The page ends with "Reporting a
  vulnerability" → `/support`.
- **Gaps:**
  - No `/.well-known/security.txt` (§10), and with allow-listed routes it needs
    its own route entry.
  - The footer on `/security` omits Support (§2,
    [UI-008](../design/UI-008-unified-footer.md) §2).
  - The page renders under the visitor's `lang` while its copy is English
    (§11).
