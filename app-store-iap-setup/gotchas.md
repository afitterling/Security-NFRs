# Gotchas — read before you start

Every trap hit while building this recipe, condensed.

## App Store Connect API

- **JWT signature encoding.** Sign ES256 with `dsaEncoding: "ieee-p1363"`. Node's
  default (DER) yields a token Apple silently rejects.
- **Availability BEFORE price.** Creating a `subscriptionPrice` before the sub has
  a `subscriptionAvailability` fails **409 `ENTITY_ERROR.RELATIONSHIP.INVALID`**,
  blaming the price point. The real cause is no territory availability. Create
  availability (all 175 territories + `availableInNewTerritories:true`) first.
- **No auto-equalization.** Setting the base-territory price prices **only** that
  territory. To go worldwide, read the base price point's `/equalizations` and
  POST a price for each of the other ~174 territories.
- **Price points are subscription-scoped and rotate.** Their ids encode
  subscription + territory + tier. Fetch fresh (`filter[territory]=…`) right before
  using them; don't cache across runs.
- **Intro offers are per-territory.** `territory` is a **required** relationship;
  one all-territory call 409s `RELATIONSHIP.REQUIRED`. Loop all 175.
- **Rate limits (HTTP 429).** ~350 POSTs for full pricing+trials per sub pair.
  Limit concurrency (~4–8), exponential backoff, and make loops **idempotent**
  (skip already-done territories) so you can re-run to fill gaps.
- **MISSING_METADATA persists** after price + localization until an **App Review
  screenshot** is uploaded per subscription (separate image upload flow).
- **Query quirks.** `subscriptionAvailability` rejects a `limit` param;
  `…/prices` rejects `fields[...]=territory` (it's an *include*, not a field).
- **Product ids are permanent.** Can't rename or fully delete — only mark
  not-for-sale. Decide the naming convention up front.
- **Localization name ≤ 30 chars.**

## RevenueCat v2 API

- **Multi-app offering sharing.** One project hosts many apps; the **current**
  offering's `$rc_monthly`/`$rc_annual` packages hold one product per app. **Add**
  your product to the existing package — creating a new current offering would
  break sibling apps. (`expand=product`, singular.)
- **`store_identifier` must equal the App Store product id** exactly, or receipts
  don't link.
- **Dashboard-only:** App Store credentials (App-Specific Shared Secret / ASC API
  key) for receipt validation, and the public `appl_…` SDK key — neither is
  settable/retrievable via the v2 API.
- **Key is project-scoped.** A `sk_…` v2 key only sees its own project; project
  creation itself is a dashboard action.

## Landing page

- **`CloudFront-Viewer-Country` is not forwarded by default** — add it to the
  origin request policy or the Lambda never sees it.
- **alpha-3 ↔ alpha-2 mismatch.** App Store territories are alpha-3; the header is
  alpha-2. Convert when generating the price file. **Kosovo: `XKS` → `XK`.**
- **Always provide a fallback country** (`US`/USD) for unknown/absent headers.
