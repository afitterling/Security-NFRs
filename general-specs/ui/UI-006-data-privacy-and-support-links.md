# UI-006 — Data-privacy link & support page

- **Status:** Proposed
- **Group:** User interface & design
- **Applies to:** All client apps (web + native).
- **Last updated:** 2026-08-26

The **routes** this spec points at are specified in their own right:
[PAGE-001](../pages/PAGE-001-privacy-route.md) (`/privacy`) and
[PAGE-003](../pages/PAGE-003-support-route.md) (`/support`). This spec governs
the **links** — that every client surfaces them, that they resolve, and that
they match the store listing.

## Requirement

1. Every app **MUST** surface, from within the app UI (settings or about), a
   reachable link to its **data-privacy policy** and a link to a **support
   page** (help / contact). Both **MUST** be reachable without signing in.
2. The privacy-policy and support targets **MUST** resolve to a live, working
   page (HTTP 200, no dead host) and **MUST** be served over **HTTPS**.
3. The same URLs **MUST** be present in the store/distribution listing where the
   platform requires them (App Store privacy-policy URL + support URL, Play Data
   safety + support contact), and **MUST** match the in-app links — one source
   of truth, no drift.
4. The links **MUST NOT** be placed behind a paywall, an account, or a
   tracking-consent gate, and **MUST NOT** leak identifiers in the URL (no
   tokens, no email, no session in the query string).
5. The support page **MUST** let the user reach support without an account. On
   submit it **MUST** send a **confirmation email** (from `no-reply@sp33c.tech`)
   to the address entered — same confirm-by-email pattern as the eject-tool —
   **MUST** forward the request to support (`info@sp33c.tech`), and **MUST** then
   show a confirmation page stating the **email was sent**. The success page
   **MUST NOT** reveal whether the address already exists (no account
   enumeration — [SEC-005](../security/SEC-005-account-enumeration.md)).
6. The privacy page **MUST** describe what the app actually does, consistent
   with the store privacy label ([PRIV-003](../privacy/PRIV-003-tracking-consent-att.md))
   and the data it collects ([PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md)).

## Rationale

A reachable privacy policy and a way to get help are baseline App Store / Play
requirements and a GDPR/DSGVO expectation. Sourcing both from one place and
checking they actually resolve turns two perennial review-rejection causes — dead
privacy URL, missing support contact — into a non-issue, and keeps the in-app
links honest against the store listing.

## Acceptance criteria

- [ ] Settings/about shows a data-privacy link and a support link, both reachable signed-out.
- [ ] Both URLs return 200 over HTTPS and are not 404 / parked.
- [ ] In-app URLs match the store-listing privacy-policy and support URLs exactly.
- [ ] No token, email, or session identifier appears in either URL.
- [ ] Submitting the support form sends a confirmation email from `no-reply@sp33c.tech` and forwards the request to `info@sp33c.tech`.
- [ ] After submit, the page shows an "email sent" confirmation that does not disclose whether the address exists.
- [ ] Privacy page content matches the store privacy label and actual data use.

## Implementation notes

- Sender/recipient addresses (`no-reply@sp33c.tech` → confirmation, `info@sp33c.tech` → support) are already configured; this spec only requires the support flow to use them.
- §5 describes the support flow's behaviour; the full route contract — the four
  addressable states, the inert `GET /support/confirm`, rate limiting on the
  public write endpoint — is [PAGE-003](../pages/PAGE-003-support-route.md).
- Related: [[PAGE-001-privacy-route]], [[PAGE-003-support-route]], [[UI-008-unified-footer]].
