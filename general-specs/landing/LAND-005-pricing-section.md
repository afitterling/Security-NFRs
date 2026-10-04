# LAND-005 — Pricing section

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** The pricing section of each paid app's landing page (`/#pricing`),
  and any other web page that lists plans.
- **Last updated:** 2026-10-04

[IAP-002](../../IAP-Subscriptions/specs/IAP-002-price-display-fidelity.md)
governs where the price *numbers* come from: generated from the store, never
typed by hand. This spec governs the *section* that shows them: which price a
visitor sees, how it is formatted, what each card must state, and how it stays
consistent with the plan catalogue and the structured data.

## Requirement

### Which price

1. **The visitor's own storefront price.** For store-sold tiers, the section
   **MUST** show the price the visitor's store would charge. It is resolved
   server-side from the request's country (CDN geo header, e.g.
   `CloudFront-Viewer-Country`) against the generated per-country table
   ([IAP-002](../../IAP-Subscriptions/specs/IAP-002-price-display-fidelity.md)
   §1). It **MUST NOT** be converted from a base currency.
2. **A deterministic fallback.** An unknown or missing country **MUST** fall back
   to one documented default storefront (US/USD, Apple's rest-of-world). It
   **MUST NOT** fall back to a blank, a `NaN` or a guess.
3. **No IP geolocation of our own.** The country comes from the CDN's header,
   resolved at the edge. The visitor's IP **MUST NOT** be logged, sent to a
   geolocation service or stored for this purpose
   ([PRIV-001](../privacy/PRIV-001-data-minimization-and-retention.md)).
4. Tiers not sold through a store (Enterprise, "contact us") **MAY** show a
   static, localized price or "Contact", taken from the catalogue.

### How it reads

5. Prices **MUST** be formatted with `Intl.NumberFormat(locale, { style:
   "currency", currency })`, using the *visitor's language* and the *storefront's
   currency*. Whole amounts drop the decimals (`$9`, not `$9.00`), and
   zero-decimal currencies (JPY, KRW) are never shown with decimals.
6. Each card **MUST** show the following:
   - the plan name;
   - the price with its period (`/mo`, `/yr`);
   - the limits that differ between plans, taken from the plan catalogue, not
     retyped
     ([`general-specs-payment/plans/`](../../general-specs-payment/plans/plan-model.md));
   - one CTA.
   Free tiers say `0` in the local currency, or "Free".
7. **At most one card is highlighted** ("Popular"), and the highlight is not the
   only way to tell plans apart. Price and limits are always text.
8. A **tax line** **MUST** state how VAT is handled ("Prices plus statutory VAT
   where applicable", or "incl. VAT"), and it **MUST** be true for the channel
   (store prices include VAT in the EU).
9. If annual pricing is offered, the annual price **MUST** be the store's own
   annual price. A "save N %" claim **MUST** be computed from the two shown
   numbers ([IAP-002](../../IAP-Subscriptions/specs/IAP-002-price-display-fidelity.md) §6).

### Consistency

10. **One catalogue.** Plan names, order, limits and CTA targets come from one
    plan definition, shared with enforcement and the in-app paywall. The static
    fallback price table (for tiers the generated table doesn't cover) **MUST**
    live next to it, not in the i18n dictionary.
11. **Structured data matches the render.** If the page emits `SoftwareApplication`
    `offers` ([SEO-004](../seo/SEO-004-structured-data.md)), each `Offer` **MUST**
    carry the same price and currency that the cards render for that request. If
    prices could not be resolved (error boundary), the offers **MUST** be
    omitted, not half-filled.
12. **The HTML must not be cached across countries.** A page whose prices depend
    on the visitor's country **MUST NOT** be shared-cached without the country
    in the key ([LAND-004](LAND-004-language-and-locale-persistence.md) §8 covers
    the same trap for the locale).

## Rationale

Showing a German visitor "$19" when the App Store will charge them "19,99 €" is
two errors at once: the wrong currency, and a number that matches nothing they
will see at checkout. Resolving the storefront at the edge and formatting it in
the visitor's language makes the landing page agree with the store sheet to the
cent. The consistency rules exist because pricing is the copy that changes most
often. Every surface that retypes a plan's price or limit is a surface that will
eventually disagree with the others, and it has already happened (see the gaps
below).

## Acceptance criteria

- [ ] Requests with `CloudFront-Viewer-Country: DE`, `JP` and `GB` show EUR, JPY (no decimals) and GBP prices that equal each store's checkout sheet.
- [ ] A request with no country header shows the documented default storefront.
- [ ] No visitor IP is logged or sent anywhere for pricing.
- [ ] Each card shows plan name, price with period, the plan's limits from the catalogue, and one CTA.
- [ ] Exactly one card is highlighted.
- [ ] A truthful VAT line is present.
- [ ] Grep finds no plan price or limit literal outside the plan catalogue and the generated price file.
- [ ] The JSON-LD offers equal the rendered card prices for the same request, and they are absent when the loader fails.
- [ ] Two requests from different countries through the CDN each get their own prices.

## Implementation notes

- **WebhookPush** is the reference:
  - `lib/pricing.ts` is generated by `scripts/gen-pricing.mjs` from App Store
    Connect subscription prices, keyed by ISO alpha-2 to match
    `cloudfront-viewer-country`, with US as the default.
  - `formatPrice` uses `Intl.NumberFormat` with dropped decimals for whole
    amounts.
  - The cards are Free · Starter · Pro (highlighted) · Enterprise (static
    price). The VAT line is `t.pricing.vat`.
  - `seo.ts` `softwareApplicationLd` takes the same geo prices and is omitted
    when the loader didn't run.
  - **Infrastructure note:** SST Remix's default origin policy already forwards
    the geo header. Adding a custom `OriginRequestPolicy` for it 403s the whole
    site.
- **Gap (§10):** `marketing/RECONCILE.md` §1 records the drift this spec
  prevents. The static `PLAN_PRICES` in `i18n.ts` showed Pro at $19/mo while
  `pricing.ts` (US) has 17.0, and older docs said €10. Limits on the cards come
  from `i18n.ts` (`p.calls`, `planWebhooks`) rather than `sst/src/lib/plans.ts`.
- **Gap (§12):** satisfied only incidentally, because every document is
  `no-store` for the locale (LAND-004 §8). If edge caching returns, the country
  has to be in the key.
- **Gap (§9):** the generated table carries `proYearly`, but the section doesn't
  show it.
