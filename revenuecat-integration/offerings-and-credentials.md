# Empty Offerings, Credentials, and the Multi-App Trap

`getOfferings()` came back empty and the paywall says *"Plans aren't available."*
This is **almost always configuration, not code** — RevenueCat surfaces it as
`CONFIGURATION_ERROR` ("None of the products… could be fetched from App Store
Connect"). Walk this checklist top to bottom.

## The empty-offerings checklist (in likelihood order)

1. **Paid Applications Agreement not active.** If the paid-apps agreement (+ tax &
   banking) isn't **Active** in App Store Connect → Business, StoreKit returns
   **zero** products — in sandbox *and* production. Not exposed via the ASC API;
   check the dashboard. This alone blocks everything downstream.
2. **Product-id mismatch.** The RC product `store_identifier` must **exactly**
   equal the App Store product id. Nilo shipped `pro_*` in RC while ASC had
   `nilo_pro_*` → empty. Store ids are **permanent**; you cannot rename in ASC, so
   fix the *other* side (RC + any local `.storekit`) to match ASC.
3. **Products on the wrong RC app** (the multi-app trap — see below).
4. **Products not `READY_TO_SUBMIT`** / missing metadata / no price / not available
   in the tester's territory.
5. **Credentials missing** (§ below) — receipts can't validate, metadata won't sync.
6. **Propagation delay.** Freshly created/priced products can take minutes to
   ~24h to be served by StoreKit. Retry later; a rebuild does **not** speed this up.
7. **A local StoreKit config file is masking the real store** — see
   `sandbox-and-gotchas.md`.

## The multi-app trap (cost us hours)

A single RC **project** hosts several **apps**: `app_store`, `play_store`,
`rc_billing` (Web Billing / Stripe), `test_store`. The device SDK key maps to
**one** app. Products, the `pro` entitlement, and the current offering's packages
must all be wired on **that** app.

Real Nilo failure: the correct `nilo_pro_*` ids existed in the project — but on the
**Web Billing** app (`app7dda4ce4d9`), while the **App Store** app the SDK talks to
(`app0588a3a726`) still carried the stale `pro_*`. Both "exist"; the device still
gets nothing. Always verify against the *App Store* app id:

```
GET …/customers                    # not it — this is per-user
GET …/apps                         # find the app whose type=app_store + your bundle_id
GET …/entitlements/<pro>/products  # each product's app_id must be THAT app
GET …/offerings/<current>/packages?expand=product   # each package's product.app_id likewise
```

(Wiring recipe — creating products, attaching to entitlement/offering — lives in
`../app-store-iap-setup/revenuecat-api.md`; don't duplicate it, reuse it.)

## Apple credentials — two distinct slots, both dashboard-only (NFR-7)

RevenueCat needs **two** different Apple keys, generated in different ASC sections.
They are *not* interchangeable and *cannot* be set via the v2 API.

| RC slot | ASC key type | Download name | Issuer id? | Purpose |
|---|---|---|---|---|
| **App Store Connect API** | Team API key (Users & Access → Integrations → App Store Connect API) | `AuthKey_*.p8` | **yes** | product import, refunds, server notifications |
| **In-App Purchase key** | In-App Purchase key (Users & Access → Integrations → In-App Purchase) | `SubscriptionKey_*.p8` | **no** | StoreKit 2 transaction validation |

Confirm both from the app object:
```
GET …/apps/<APP_ID>
→ { app_store: { app_store_connect_api_key_configured: true,
                 subscription_key_configured: true } }
```
"I already uploaded a key" is usually only **one** of the two slots. `.p8` private
keys download **once** at creation — Apple has no API to mint them (the API
authenticates *using* one), so this step is unavoidably manual.

## API keys: legacy v1 vs v2

- **v2 secret** (`sk_…`, created "Secret key (v2)") — the management API
  (`/v2/projects/…`: apps, products, entitlements, offerings, customers). Server-
  only; never ship it.
- **Legacy v1** keys **403 on `/v2`** with *"trying to use a legacy API key to
  access API v2."* If you get that, mint a v2 key. (The v1 REST API is still what a
  backend uses to *read* a subscriber's entitlement, e.g. Nilo's landing check.)
- **Public SDK key** (`appl_…`) — goes in the app; **not retrievable via the API**,
  copy from the dashboard into the app env (Nilo: `REVENUECAT_IOS_KEY`).
- Rotate any `sk_…` pasted into a chat/log afterwards.

## Prices live in the store, not RevenueCat

RevenueCat holds **no prices**. Set/inspect them in App Store Connect (see
`../app-store-iap-setup/app-store-connect-api.md`). RC just references the product;
the paywall renders the store's localized price at runtime.
