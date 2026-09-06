# PAGE-002 — The `/imprint` route (Impressum)

- **Status:** Proposed
- **Group:** Public pages & routes
- **Applies to:** Every app's public web surface. Mandatory for any site reachable
  from Germany/the EU that is not purely personal or family use — which is all of
  them ([apps in scope](../../general-specs/README.md#apps-in-scope)).
- **Last updated:** 2026-08-26

Until now the imprint existed only as a clause inside other specs — "the footer
must link one" ([UI-008](../design/UI-008-unified-footer.md) §2), "an imprint must
be reachable" ([PRIV-002](../privacy/PRIV-002-gdpr-dsgvo-user-rights.md) §1,
[PAGE-001](PAGE-001-privacy-route.md) §4). Those say a link must exist; none of
them says **what the page has to contain**. This spec does.

## Requirement

### Route & reachability

1. A **legal notice (Impressum)** **MUST** be served at a stable path —
   `/imprint` — on the canonical production host, over **HTTPS**, returning
   **HTTP 200**, and **MUST NOT** change between releases. `/impressum` **MUST**
   resolve to the same content (301 to `/imprint`, or serve it directly); a
   German visitor guessing `/impressum` **MUST NOT** get a 404.
2. The page **MUST** be reachable from **every** page of the public site in at
   most **two clicks**, via a footer link on the unified footer
   ([UI-008](../design/UI-008-unified-footer.md) §1–2).
3. The link **MUST** be labelled unambiguously — **"Imprint"** / **"Impressum"**
   (or "Legal notice"). A link labelled only "Contact", "About", or an icon
   **MUST NOT** be the sole route to it: German case law treats
   *Erkennbarkeit* — the label making the destination obvious — as part of the
   duty, and an unrecognisable label is the classic *Abmahnung* trigger.
4. The page **MUST** render for an anonymous visitor: no account, no paywall, no
   cookie/consent gate, no JS-only render path
   ([UI-008](../design/UI-008-unified-footer.md) §5). It **MUST NOT** carry any
   identifier in the URL ([AUTH-002](../authentication/AUTH-002-no-tokens-in-urls.md)).
5. The imprint and the privacy notice **MUST** be **separate pages**, each
   linked from the other ([PAGE-001](PAGE-001-privacy-route.md) §4). Burying the
   imprint as a section of the privacy policy does not discharge the duty.
6. Where a native client exists, the in-app about/settings screen **MUST** link
   the same absolute URL rather than reproducing the text
   ([UI-006](../ui/UI-006-data-privacy-and-support-links.md) §1).
7. The URL **MUST** come from one exported constant (`IMPRINT_URL`, alongside
   `PRIVACY_URL` / `SUPPORT_URL`), consumed by the footer, the in-app link, and
   any store metadata field that takes it — one source of truth, no per-surface
   copies.

### Content

8. The page **MUST** state, in a form a visitor can read without downloading
   anything (no PDF-only imprint, no image of text):
   - the **provider's full legal name and legal form** (for a company: as
     registered — e.g. `… GmbH`, `… UG (haftungsbeschränkt)`);
   - a **physical postal address** — street, number, postcode, city, country.
     A P.O. box or a c/o packet-service address **MUST NOT** be used;
   - **contact details** enabling rapid electronic contact: a working
     **email address** and at least one **second means** of direct
     communication (phone number, or a contact form that reaches the same
     inbox — [PAGE-003](PAGE-003-support-route.md));
   - where the provider is registered: the **register court and number**
     (Handelsregister/Vereinsregister/…);
   - the **VAT identification number** (USt-IdNr., § 27a UStG) **if one has
     been issued** — it **MUST NOT** be invented or replaced by the tax number;
   - for a legal entity: the **authorised representatives** (Geschäftsführer /
     Vorstand).
9. Where the site carries **journalistic-editorial content** (a blog, news, a
   changelog written as editorial), the page **MUST** additionally name the
   person **responsible for content** with their own full address
   (*Verantwortlicher i.S.d. § 18 Abs. 2 MStV*).
10. Where the activity is **supervised or requires authorisation** (regulated
    profession, licensed trade), the page **MUST** name the competent
    **supervisory authority** and the relevant chamber/professional rules.
11. The page **MUST** carry the **EU online dispute resolution / consumer
    arbitration** statement where the app sells to consumers — at minimum
    whether the provider is willing or obliged to participate in dispute
    resolution before a consumer arbitration board (§ 36 VSBG).
12. Every statement on the page **MUST** be **accurate and current**. An address
    or representative that has changed **MUST** be updated in the same release
    as the change — a stale imprint is a live liability, not a documentation
    debt.
13. The email address **MUST** be a real, monitored mailbox and **MUST NOT** be
    obfuscated into unreachability (image-only, JavaScript-assembled, or
    `name [at] domain [dot] tld`). Spam avoidance is not a defence against the
    "rapid electronic contact" requirement — use `info@sp33c.tech` behind a
    plain `mailto:` and filter server-side.
14. The page **MUST NOT** contain the widely copy-pasted "Haftungsausschluss
    LG Hamburg 1998" disclaimer or comparable boilerplate. It is legally
    inoperative and signals a copied imprint.

### Rendering, indexing, availability

15. The page **MUST** be reachable in every baseline locale; the **legal
    identity data MUST NOT be translated or transliterated** (a registered name
    and address are identifiers, not copy). Surrounding labels **SHOULD** be
    localized ([I18N-001](../internationalization/I18N-001-localization.md)).
16. The page **MUST** stay readable at every breakpoint and at increased font
    scale, with headings in order
    ([UI-001](../design/UI-001-responsive-layout.md),
    [A11Y-001](../accessibility/A11Y-001-baseline.md)).
17. Indexing posture **MUST** be consistent across `robots.txt`, `sitemap.xml`,
    `rel="canonical"` and `<meta robots>` — the same either/or as
    [PAGE-001](PAGE-001-privacy-route.md) §17. The imprint **MUST NOT** be
    hidden via `robots.txt` ([SEO-001](../seo/SEO-001-robots-txt.md) §4): it is
    public by legal design.
18. The route **MUST** be static — no database, session, or third-party call
    required to render ([REL-002](../reliability/REL-002-resilience-and-failure-modes.md)) —
    and **SHOULD** be covered by the same uptime check as `/privacy`
    ([REL-001](../reliability/REL-001-observability-and-alerting.md)).
19. Non-production copies **MUST NOT** be indexable
    ([SEO-001](../seo/SEO-001-robots-txt.md) §5,
    [DEL-002](../delivery/DEL-002-environments-and-promotion.md)).

## Rationale

The imprint duty (§ 5 DDG, formerly § 5 TMG; § 18 MStV for editorial content) is
one of the few web requirements enforced by **private parties for profit**: a
competitor's lawyer sends a cease-and-desist, and the cost lands before any
regulator is involved. The failure modes are dull and entirely preventable —
missing page, unrecognisable link label, P.O. box instead of a street address,
an obfuscated email, a copied disclaimer, an address that moved two years ago.

Writing it as a spec rather than a footer clause means the *content* is checkable
in review, and means the page gets the same route guarantees as `/privacy`: one
constant, one path, reachable signed-out, alive under dependency failure. App
Review also checks the support/legal surface of a listing; a 404 here is the same
class of release blocker as a dead privacy URL.

## Acceptance criteria

- [ ] `GET https://<prod-host>/imprint` → 200, HTTPS, anonymous, no consent gate.
- [ ] `GET /impressum` resolves to the same content (200 or 301 → `/imprint`), never 404.
- [ ] Every public page's footer links it, labelled "Imprint"/"Impressum", within two clicks.
- [ ] The imprint is its own page, linked from `/privacy`, and `/privacy` is linked from it.
- [ ] Page states legal name + form, street address (not a P.O. box), email, a second contact means, register court + number, VAT ID if issued, and representatives.
- [ ] Editorial-content responsible person named with address, where applicable.
- [ ] Supervisory authority / chamber named, where applicable.
- [ ] Consumer dispute-resolution statement present where the app sells to consumers.
- [ ] Email is a plain `mailto:` to a monitored mailbox — not an image, not JS-assembled.
- [ ] No "LG Hamburg" or comparable copy-pasted disclaimer.
- [ ] Legal identity data is untranslated in every locale; page readable at 320px and 200% font scale.
- [ ] `IMPRINT_URL` is a single exported constant; footer, in-app, and store links all derive from it.
- [ ] Indexing posture consistent across robots/sitemap/canonical/meta; route not `Disallow`-ed.
- [ ] Route renders with the datastore unreachable; preview/staging copies are `noindex`.

## Implementation notes

- Ship it next to `/privacy` as a sibling static route — same layout component,
  same footer, same uptime check. The two pages differ in copy, not in
  machinery, so build them in one pass.
- The content in §8–§11 is a **legal** determination for the operating entity,
  not a per-app one: draft it once for `sp33c.tech`, then have every app render
  the same block. What this spec constrains is that the page exists, resolves,
  is labelled findably, and is kept current — not the drafting.
- **§12 is the one that rots.** Tie the imprint's review to the same trigger as
  the privacy notice's "last updated" date ([PAGE-001](PAGE-001-privacy-route.md) §9):
  entity details change rarely, so nothing routine will catch a stale address.
- **Per-app status:** **Gap — unverified across all apps.** No app in scope has
  a confirmed `/imprint` route; `UI-008` and `PRIV-002` require the link but the
  pages were never audited. Audit before adopting.
- Related: [[PAGE-001-privacy-route]], [[PAGE-003-support-route]],
  [[UI-008-unified-footer]], [[PRIV-002-gdpr-dsgvo-user-rights]].
