# UI-008 — Unified footer with legal & informational links

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** Each app's public landing / marketing site and its sub-pages (web).
- **Last updated:** 2026-07-18

## Requirement

1. Every landing page and every one of its sub-pages **MUST** render a single,
   **unified footer** — the same footer component, in the same place, on every
   page. There **MUST NOT** be per-page footer variants that drift.
2. The footer **MUST** contain a reachable **imprint / legal notice**
   (Impressum) and links to the **privacy policy** and **support / contact**
   (one source of truth with the in-app links required by
   [UI-006](../ui/UI-006-data-privacy-and-support-links.md)). Where a **terms /
   EULA** page exists it **MUST** be linked here too.
3. The footer **SHOULD** also carry a link to the home route (consistent with the
   navigable header, [UI-007](UI-007-navigable-landing-header.md)) and a
   copyright / legal-entity line.
4. Every footer link **MUST** be a real link (`<a href>` / framework `Link`) —
   keyboard-focusable and crawlable, not an `onClick`-only element — and its
   target **MUST** resolve to a live page (HTTP 200, HTTPS, no 404 / parked host).
5. All footer targets **MUST** be reachable **without signing in** and **MUST
   NOT** sit behind a paywall, account, or tracking-consent gate, and **MUST
   NOT** leak identifiers in the URL (no tokens, email, or session in the query
   string).
6. Footer links **MUST** have accessible names and the footer **SHOULD** be
   consistent in placement and appearance across all pages
   ([UI-002](UI-002-visual-design-language.md), [A11Y-001](../accessibility/A11Y-001-baseline.md)).

## Rationale

An imprint/Impressum and a reachable privacy policy are legal baseline
requirements (German TMG/DDG imprint duty, GDPR/DSGVO), and the footer is where
visitors expect to find them. Sourcing one unified footer from a single component
— rather than hand-rolling it per page — stops the usual drift where some pages
carry the legal links and others silently omit them, which is both a compliance
risk and a store-review rejection cause.

## Acceptance criteria

- [ ] The same footer component renders on the landing page and every sub-page.
- [ ] Footer contains an imprint/Impressum link plus privacy and support/contact links.
- [ ] Terms/EULA is linked when such a page exists; a copyright/entity line is present.
- [ ] Every footer link is a real anchor, returns 200 over HTTPS, and is not 404/parked.
- [ ] All targets are reachable signed-out, behind no paywall/consent gate, with no identifier in the URL.
- [ ] Footer links have accessible names; placement is consistent across pages.

## Implementation notes

- Status `Proposed`: adopt when each app's landing site is next revised; audit
  existing pages for any missing or divergent footer and unify them.
- Shares its privacy/support targets with [UI-006](../ui/UI-006-data-privacy-and-support-links.md)
  (one source of truth) and complements [UI-007](UI-007-navigable-landing-header.md)
  (header home link) — header and footer together bracket every page's navigation.
