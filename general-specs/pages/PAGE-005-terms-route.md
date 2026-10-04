# PAGE-005 — The `/terms` route (Terms of Use / EULA)

- **Status:** Proposed
- **Group:** Public pages & routes
- **Applies to:** Every app that sells an auto-renewable subscription or another
  in-app purchase, and every app with accounts. Apps with neither **MAY** rely on
  Apple's standard EULA, but still **MUST** say so in the store metadata.
- **Last updated:** 2026-10-04

[UI-008](../design/UI-008-unified-footer.md) §2 says a terms page "**MUST** be
linked where it exists". App Review makes it exist for every app with
subscriptions: Guideline 3.1.2 requires a functional link to the Terms of Use
(EULA) in the app's metadata **and** inside the app's purchase flow. This spec is
the route contract.

## Requirement

### Route & reachability

1. The terms **MUST** be served at a stable path, `/terms`, on the production
   host. The response is HTTPS, `200` and server-rendered, for an anonymous
   visitor with no consent gate. The path **MUST NOT** change once filed with a
   store ([LAND-011](../landing/LAND-011-not-found-scanners-and-legacy-redirects.md) §9).
2. The same absolute URL, from one exported `TERMS_URL`, **MUST** be linked from:
   - the unified footer of every public page
     ([UI-008](../design/UI-008-unified-footer.md) §2);
   - the in-app paywall, next to the purchase button, together with the privacy
     link (Guideline 3.1.2);
   - the in-app settings/about screen;
   - the App Store metadata: either the description, or the custom EULA field in
     App Store Connect.
3. The page links back to `/`, to the privacy notice
   ([PAGE-001](PAGE-001-privacy-route.md)) and to the imprint
   ([PAGE-002](PAGE-002-imprint-route.md)).

### Content

4. The page **MUST** contain at minimum the following:
   - the parties: the provider by legal name, and a statement that the agreement
     is with the provider, not Apple;
   - the license grant and the restrictions;
   - a description of the service;
   - **subscription terms** (below);
   - minimum age;
   - acceptable use, and how to report abuse;
   - maintenance and support (provider, not Apple);
   - warranty;
   - product claims (provider responsible);
   - intellectual-property claims;
   - limitation of liability;
   - **Apple as third-party beneficiary**;
   - privacy (link);
   - governing law;
   - contact.
   Together these cover Apple's *Minimum Terms of Developer's End-User License
   Agreement* (DPLA Schedule 2, Exhibit B).
5. The **subscription section** **MUST** state:
   - the plan names;
   - that subscriptions auto-renew for the same period unless cancelled at least
     24 hours before the period ends;
   - that renewal is charged within the 24 hours before the period ends;
   - where to manage or cancel (Apple ID settings, or the web billing portal for
     web purchases);
   - that cancellation takes effect at the end of the current period;
   - how free trials convert;
   - that prices are shown in local currency before purchase.
   Each statement **MUST** match the store configuration
   ([IAP-002](../../IAP-Subscriptions/specs/IAP-002-price-display-fidelity.md),
   [IAP-004](../../IAP-Subscriptions/specs/IAP-004-payment-gating-freemium-throttling-sync.md)).
6. If plans have usage limits, the terms **MUST** say that limits apply and are
   enforced automatically, and **MUST** point at where they are listed (the
   pricing section). The terms **MUST NOT** restate the numbers, which drift
   ([LAND-012](../landing/LAND-012-truthful-marketing-claims.md) §4).
7. The page **MUST** carry a visible **"last updated"** date. A material change
   **SHOULD** be announced to existing users before it takes effect.
8. The terms **MUST NOT** contradict the privacy notice, the security page or the
   store listing. Where they touch the same subject (data deletion, refunds), one
   of them is authoritative and the others link to it.

### Language & indexing

9. The terms **MAY** be English-only
   ([LAND-004](../landing/LAND-004-language-and-locale-persistence.md) §7). An
   English-only page declares `lang="en"`. Apps whose consumers are mainly in
   Germany **SHOULD** offer a German version, because consumer-facing terms in a
   language the consumer doesn't understand weaken their enforceability.
10. The page is indexable and listed in the sitemap
    ([SEO-002](../seo/SEO-002-sitemap.md)), with its own title and description
    ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md)).

## Rationale

The most common App Review rejection for a subscription app is a missing or
non-functional EULA link (Guideline 3.1.2). The second is subscription copy that
doesn't match what the store sells. A stable route, one URL constant shared by
the paywall, the footer and the metadata, and a subscription section checked
against the store configuration remove both. Restating limits in the terms only
creates a second place for the numbers to be wrong.

## Acceptance criteria

- [ ] `/terms` returns 200 signed-out over HTTPS, server-rendered, and is in the sitemap.
- [ ] The footer of every page, the paywall, the settings screen and the App Store metadata all link the same `TERMS_URL`.
- [ ] The page contains every section in §4, including "Apple as third-party beneficiary".
- [ ] The subscription section states auto-renewal, the 24-hour window, where to cancel, when cancellation takes effect and how trials convert, and each statement matches App Store Connect.
- [ ] The terms contain no limit numbers, only a pointer to pricing.
- [ ] A "last updated" date is visible.
- [ ] An English-only page declares `lang="en"`.

## Implementation notes

- **WebhookPush** (`webhook/sst/landing/app/routes/terms.tsx`) is the reference.
  It has 14 sections, from "This agreement" to "Governing law", including Apple
  as third-party beneficiary. It is English-only by design, with the provider
  block taken from `PROVIDER` in `i18n.ts`, and is linked from the landing footer
  and the paywall.
- **Gaps:**
  - There is no "last updated" date (§7).
  - The page renders under the visitor's `lang` (§9).
  - It has no unified footer and no nav, only a back link (§2,
    [UI-008](../design/UI-008-unified-footer.md) §1).
