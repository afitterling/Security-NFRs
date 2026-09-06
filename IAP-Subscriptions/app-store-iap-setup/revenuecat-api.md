# RevenueCat v2 API — Wiring the Products

Base URL: `https://api.revenuecat.com/v2`. Auth: `Authorization: Bearer sk_…`
(a **v2 secret key**, created at RevenueCat → Project settings → API keys →
+ New → "Secret key (v2)", **write** access). Never ship `sk_…`; it is server-only.

Worked example: project `sp33c`, new app `threethings` (bundle `tech.sp33c.three`),
products mapped to the App Store product ids created in
[app-store-connect-api.md](app-store-connect-api.md).

Minimal node helper:
```js
const r = await fetch('https://api.revenuecat.com/v2'+path, {
  method, headers: { Authorization:'Bearer '+process.env.RC_KEY,
                     'Content-Type':'application/json' },
  body: body && JSON.stringify(body) });
```

## 1. Discover project + apps

```
GET /projects                                  → pick the project id (proj…)
GET /projects/<PROJECT_ID>/apps                → existing apps; note app types
```
A v2 secret key is scoped to **one project**. Apps inside a project can be
`app_store`, `play_store`, `rc_billing` (web), `test_store`.

## 2. Create the App Store app

```
POST /projects/<PROJECT_ID>/apps
{ "name":"threethings", "type":"app_store", "app_store": { "bundle_id":"tech.sp33c.three" } }
```
Returns `app…` id. Note the response shows
`app_store_connect_api_key_configured:false` and `subscription_key_configured:false`
— see *Dashboard-only* below.

## 3. Create products (one per App Store subscription)

```
POST /projects/<PROJECT_ID>/products
{ "app_id":"<APP_ID>",
  "store_identifier":"tech.sp33c.three.premium.monthly",  // = the ASC productId
  "type":"subscription",
  "display_name":"Premium Monthly" }
```
Repeat for the annual. `store_identifier` **must** equal the App Store product id
or RevenueCat can't link receipts.

## 4. Entitlement + attach products

```
POST /projects/<PROJECT_ID>/entitlements
{ "lookup_key":"premium", "display_name":"Premium" }

POST /projects/<PROJECT_ID>/entitlements/<ENT_ID>/actions/attach_products
{ "product_ids": ["<MONTHLY_PROD_ID>", "<ANNUAL_PROD_ID>"] }
```
`lookup_key` must match what the app checks (`PREMIUM_ENTITLEMENT = "premium"`).

## 5. Offering & packages — the multi-app pattern (key insight)

A single RevenueCat project commonly hosts **several different apps**. There is one
**current** offering per project (`is_current: true`, usually `lookup_key:"default"`).
Its standard packages — `$rc_monthly`, `$rc_annual`, `$rc_lifetime` — each hold
**one product per app**, and the SDK serves whichever product matches the running
app's bundle. So:

> **Add** your product into the existing current offering's package — do **not**
> create a new current offering. This is non-destructive: sibling apps (e.g. `Nilo`)
> keep their products in the same package and are unaffected.

```
GET  /projects/<PROJECT_ID>/offerings                          # find is_current + its id
GET  /projects/<PROJECT_ID>/offerings/<OFFERING_ID>/packages   # find $rc_monthly / $rc_annual ids
GET  /projects/<PROJECT_ID>/packages/<PKG_ID>?expand=product   # inspect existing products

POST /projects/<PROJECT_ID>/packages/<MONTHLY_PKG_ID>/actions/attach_products
{ "products": [ { "product_id":"<MONTHLY_PROD_ID>", "eligibility_criteria":"all" } ] }
# repeat: annual product → $rc_annual package
```
Verify: re-`GET …/packages/<id>?expand=product` and confirm your new
`store_identifier` sits alongside the sibling apps' products.

(`expand` is singular: `expand=product` for packages, not `products`.)

## Dashboard-only (the v2 API cannot do these)

1. **App Store credentials for receipt validation** — the **App-Specific Shared
   Secret** and/or an **App Store Connect API key** (you can reuse the same `.p8`).
   Configure under RevenueCat → Project → Apps → <app> → App Store. Until done,
   purchases won't validate and the `premium` entitlement never activates.
2. **Public SDK key** (`appl_…`) — not exposed by the v2 API. Copy from the
   dashboard. With the modern key system this is per-project/platform and works
   for every App Store app in the project.

## Repo wiring (this stack)

- **Backend** (`sst/src/revenuecat.ts`): reads entitlement via the v1 REST API
  using `REVENUECAT_SECRET` (a **v1 secret** read key) — set it in
  `sst/.env.prod` (currently empty). Powers the landing page's "is this user
  premium?" check; degrades to inactive if unset.
- **iOS app** (`iosapp`): `REVENUECAT_IOS_KEY` (the **public `appl_`** key) in
  `.env.dev` / `.env.prod`, surfaced via `app.config.js` → `extra.revenuecatIosKey`
  → `Purchases.configure`. The app reads `offerings.current` and the `premium`
  entitlement; product ids are **not** hardcoded in the app — they come from the
  offering, so the package wiring in step 5 is what makes the paywall populate.
