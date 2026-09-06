# PAGE-001 — The `/privacy` route

- **Status:** Proposed
- **Group:** Public pages & routes
- **Applies to:** Every app's public web surface. The native clients link *to*
  this route rather than reimplementing it ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §1).
- **Last updated:** 2026-08-26
- **History:** Drafted as bundle-local `PRIV-004` in `privacy-page/route/`;
  promoted here when the `PAGE` group was created. `PRIV-004` is **retired
  unused** and MUST NOT be reassigned.

This spec is about the **route itself** — where it lives, that it resolves, and
that its claims match the shipped system. The cross-cutting requirements that
happen to land on it live in their own groups and are linked from here.

## Requirement

### Route & reachability

1. The privacy notice **MUST** be served at a **stable, guessable path** —
   `/privacy` — on the canonical production host, over **HTTPS**, returning
   **HTTP 200**. The path **MUST NOT** change between releases; if a legacy path
   exists (`/datenschutz`, `/privacy-policy`) it **MUST** 301 to `/privacy`, so
   the URL filed with the App Store never rots
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §2–3).
2. The route **MUST** render for an anonymous visitor: no account, no paywall,
   no cookie/tracking-consent gate, no JS-only render path
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §4,
   [UI-008](../design/UI-008-unified-footer.md) §5).
3. The URL **MUST** carry no identifier — no token, email, user id, or session in
   the path or query string ([AUTH-002](../authentication/AUTH-002-no-tokens-in-urls.md)).
   `?lang=` is the only query parameter this route accepts
   ([I18N-001](../internationalization/I18N-001-localization.md) §4).
4. An **imprint / legal notice** (Impressum) **MUST** be reachable as a sibling
   route and linked from the privacy page, and the privacy page **MUST** be
   linked from the unified footer of every page
   ([PAGE-002](PAGE-002-imprint-route.md),
   [PRIV-002](../privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §1,
   [UI-008](../design/UI-008-unified-footer.md) §2).
5. The same absolute URL **MUST** be used in the footer, in the in-app
   settings/about link, and in the store listing's privacy-policy field — **one
   constant, one source of truth**, no per-surface copies
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §3).

### Content

6. The page **MUST** state, at minimum: **who** the controller is (entity +
   contact), **what** personal data is collected and **why**, the **lawful
   basis** per use, **retention** periods, **which third parties** receive data
   (processors, and any egress from a feature), how to exercise **user rights**,
   and the **tracking posture** of the app.
   Each of those statements **MUST** be true of the shipped system —
   [PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md) §5 (disclose
   third-party egress), [PRIV-002](../privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §4
   (lawful basis), [PRIV-003](../privacy/PRIV-003-tracking-consent-att.md) §5
   (label matches reality).
7. The retention statement **MUST** match the actual TTLs configured on the
   stores ([PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md) §2–3).
   A notice that promises expiry the tables do not implement is a false statement,
   not a rounding error.
8. The tracking section **MUST** agree with the store privacy label and the
   shipped binary. For a no-tracking app it **MUST** state that no advertising
   identifier is collected and no data is shared with data brokers — and the
   binary **MUST** back that up (no ATT string, no `collectDeviceIdentifiers()`)
   → [non-tracking-purchases](../../IAP-Subscriptions/non-tracking-purchases/README.md).
9. The page **MUST** carry a visible **"last updated" date** and that date
   **MUST** be changed whenever the copy changes. Material changes **SHOULD** be
   summarised (a short change note or a dated revision list).
10. The page **MUST NOT** claim data practices the app does not have (a
    copy-pasted generic policy mentioning analytics, ad networks, or cookies the
    app doesn't use is as wrong as omitting one it does).

### User rights, made actionable

11. The rights section **MUST** name a **working way to exercise each right** —
    erasure, access/export, consent withdrawal — as a link or an address, not
    prose alone: either an in-app entry point or the support route
    ([PAGE-003](PAGE-003-support-route.md),
    [PRIV-002](../privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §2–4,
    [UI-006](../ui/UI-006-data-privacy-and-support-links.md) §5).
12. Where the page exposes a **request form** (deletion/export by email), that
    form is a public write endpoint and **MUST** inherit the form requirements:
    rate limiting ([SEC-002](../security/SEC-002-rate-limiting-and-lockout.md),
    [SEC-006](../security/SEC-006-edge-rate-limiting.md)), request integrity
    ([SEC-003](../security/SEC-003-request-integrity-csrf.md)), and a response
    that **MUST NOT** disclose whether the address is registered
    ([SEC-005](../security/SEC-005-account-enumeration.md)). Prefer routing such
    requests through the existing support flow ([PAGE-003](PAGE-003-support-route.md))
    rather than adding a second endpoint.

### Rendering

13. The page **MUST** be localized from the catalog in all baseline locales and
    auto-detect the visitor's language
    ([I18N-001](../internationalization/I18N-001-localization.md) §2–4). Legal
    copy **MUST NOT** be machine-translated without review, and a locale whose
    review is pending **MUST** fall back to a reviewed language rather than ship
    an unreviewed translation.
14. Long-form legal text **MUST** stay readable at every breakpoint and at
    increased font scale — no fixed-height scroll boxes, no horizontal overflow,
    body text **SHOULD** sit in a bounded measure (~65–75 characters)
    ([UI-001](../design/UI-001-responsive-layout.md),
    [UI-004](../design/UI-004-usability-baseline.md),
    [A11Y-001](../accessibility/A11Y-001-baseline.md) §1/§3).
15. Section headings **MUST** use real heading elements in order (`h1` → `h2`),
    so the page is navigable by screen reader and skimmable by everyone
    ([A11Y-001](../accessibility/A11Y-001-baseline.md)).
16. The page **MUST** render without any consent-gated third-party script.
    Because it must be reachable *before* a consent decision
    ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §4), no analytics,
    tag manager, or embedded font/CDN call that itself requires consent may be a
    prerequisite for the content appearing.

### Indexing

17. The site **MUST** pick one indexing posture for the route and apply it
    consistently across the whole SEO surface:
    - **Indexable (default):** unique localized `<title>` + description, absolute
      HTTPS `rel="canonical"`, listed in `sitemap.xml`, not `Disallow`-ed in
      `robots.txt` ([SEO-002](../seo/SEO-002-sitemap.md) §3–4,
      [SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §1–2).
    - **Non-indexable:** `<meta name="robots" content="noindex">`, **absent** from
      the sitemap, and social/canonical tags omitted rather than misleading
      ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §8).

    A page that is `noindex` **but** listed in the sitemap, or canonical-tagged
    **but** `Disallow`-ed, is the failure this rule exists to prevent.
18. `robots.txt` **MUST NOT** be used to hide anything here — the privacy notice
    is public by design ([SEO-001](../seo/SEO-001-robots-txt.md) §4).
19. Structured data **MAY** be omitted; if emitted it **MUST** describe visible
    on-page content only ([SEO-004](../seo/SEO-004-structured-data.md) §2).

### Availability

20. The route **MUST** survive the failure of every non-essential dependency —
    it is static content and **MUST NOT** require a database, session lookup, or
    third-party call to render
    ([REL-002](../reliability/REL-002-resilience-and-failure-modes.md)).
21. The URL **SHOULD** be covered by an uptime/status check that alerts on
    non-200, because a dead privacy URL is both a compliance failure and a
    store-review rejection cause
    ([REL-001](../reliability/REL-001-observability-and-alerting.md),
    [UI-006](../ui/UI-006-data-privacy-and-support-links.md) §2).
22. Non-production copies of the route **MUST NOT** be indexable
    ([SEO-001](../seo/SEO-001-robots-txt.md) §5,
    [DEL-002](../delivery/DEL-002-environments-and-promotion.md)); the store
    listing and in-app links **MUST** point at production only.

## Rationale

A privacy notice fails in two directions, and both are cheap to prevent and
expensive to discover late. It fails **legally** when the copy and the system
disagree — retention prose that no TTL implements, an undisclosed processor, a
"we don't track" line contradicted by an IDFA call. It fails **operationally**
when the URL rots: App Review checks that the privacy-policy URL resolves, and a
404 there blocks the release regardless of how good the copy is.

Pinning the route to one constant path, deriving the in-app and store links from
that same constant, and asserting the content against what the code actually does
turns both failure modes into checks that run before submission instead of
findings that arrive with a rejection.

## Acceptance criteria

- [ ] `GET https://<prod-host>/privacy` → 200, HTTPS, anonymous, no consent gate.
- [ ] Legacy privacy paths 301 to `/privacy`; no path in use anywhere returns 404.
- [ ] Footer link, in-app settings link, and the store-listing privacy URL are byte-identical.
- [ ] Imprint/Impressum reachable and linked from the page ([PAGE-002](PAGE-002-imprint-route.md)).
- [ ] Page states controller, data collected, purposes, lawful basis, retention, third parties, rights, tracking posture.
- [ ] Every retention period stated matches a TTL that is actually configured.
- [ ] Tracking section, store privacy label, and shipped binary agree (`expo config --introspect | grep -c NSUserTracking` → `0` for a no-tracking app).
- [ ] A visible "last updated" date is present and changes with the copy.
- [ ] Each user right names a working link or address; any request form is rate-limited and non-enumerating.
- [ ] Renders in every baseline locale, auto-detected; no unreviewed legal translation ships.
- [ ] No horizontal overflow at 320px; readable at 200% font scale; headings in order.
- [ ] Content appears with third-party/consent-gated scripts blocked.
- [ ] Indexing posture is consistent across `robots.txt`, sitemap, canonical, and `<meta robots>`.
- [ ] Route renders with the datastore unreachable; an uptime check alerts on non-200.
- [ ] Preview/staging copies are `noindex` and are not the URL filed with the stores.

## Implementation notes

- Keep the path in one exported constant (e.g. `PRIVACY_URL` next to
  `SUPPORT_URL` and `IMPRINT_URL`) consumed by the footer, the app's settings
  screen, and whatever fills the store metadata — §5 holds by construction
  rather than by review. See [PAGE-002](PAGE-002-imprint-route.md) §7.
- Legal wording is not an engineering decision. This spec constrains **where the
  page lives, that it is reachable, and that its claims match the system** — it
  does not draft the notice. Have the copy reviewed; the code-side job is
  ensuring no statement in it is contradicted by the implementation.
- Where an app has no separate settings screen, the in-app link required by
  [UI-006](../ui/UI-006-data-privacy-and-support-links.md) §1 still has to exist
  somewhere reachable signed-out (about screen, onboarding footer).
- Related: [[PAGE-002-imprint-route]], [[PAGE-003-support-route]],
  [[UI-006-data-privacy-and-support-links]],
  [[PRIV-002-gdpr-dsgvo-user-rights]], [[UI-008-unified-footer]].
