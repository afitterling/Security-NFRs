# RevenueCat integration

The single integration that covers the Expo app **and** the SST web landing.

## 1. Dashboard setup (one-time)

1. Create a RevenueCat **project**; add three apps: **App Store**, **Play
   Store**, **Web Billing** (Stripe connected).
2. Create **products** in App Store Connect / Play Console (auto-renewable subs),
   and matching **Web Billing** products. Import them into RevenueCat.
3. Create **Entitlements**: `starter`, `pro`, `enterprise`. Attach each tier's
   monthly + annual products.
4. Create an **Offering** (`default`) with packages per tier so the apps fetch
   what to show dynamically (prices come from the stores — never hardcode).
5. API keys: a **public SDK key per platform** (safe to ship) + a **secret
   server key** (Api only) + a **webhook auth header secret**.

## 2. Mobile app (`webhook-app`, Expo)

```bash
npx expo install react-native-purchases
# Expo: requires a dev/prod build (config plugin), not Expo Go.
```

```ts
import Purchases from "react-native-purchases";

// On launch, after auth — tie RC identity to OUR userId (email):
await Purchases.configure({ apiKey: RC_PUBLIC_KEY_IOS_OR_ANDROID });
await Purchases.logIn(userId);

// Plan picker:
const offerings = await Purchases.getOfferings();
const pkg = offerings.current?.availablePackages.find(p => p.identifier === "pro_monthly");
const { customerInfo } = await Purchases.purchasePackage(pkg);
const plan = entitlementToPlan(customerInfo.entitlements.active); // "pro"
```

- **Restore purchases** button → `Purchases.restorePurchases()` (App Store requirement).
- The app should **not** trust the device for enforcement — it reads `user.plan`
  from the Api for display; the Api is set by the webhook (below).

## 3. Web (`sst/landing`, Remix)

```bash
npm i @revenuecat/purchases-js
```

```ts
import { Purchases } from "@revenuecat/purchases-js";
const rc = Purchases.configure(RC_WEB_BILLING_PUBLIC_KEY, userId);
const offerings = await rc.getOfferings();
// Render packages -> on click, RC opens the Stripe-backed checkout:
await rc.purchase({ rcPackage: offerings.current.availablePackages[0] });
```

- Add routes (manual, in `vite.config.ts`): `/billing` (plan picker) and
  `/billing/return` (post-checkout). See [../ui/plan-selection.md](../ui/plan-selection.md).
- Web purchases flow through RC Web Billing (Stripe). Subscription management
  (cancel/update card) uses the RC customer portal link.

## 4. Backend webhook (SST Api) — the authoritative path

RevenueCat → `POST /billing/revenuecat/webhook` on the Api.

```ts
// Verify the Authorization header matches RC_WEBHOOK_SECRET, then:
const e = body.event;                 // INITIAL_PURCHASE, RENEWAL, CANCELLATION,
                                       // EXPIRATION, PRODUCT_CHANGE, BILLING_ISSUE...
const userId = e.app_user_id;         // == our userId, from Purchases.logIn()
const plan = entitlementToPlan(e.entitlement_ids ?? []);
const status = mapStatus(e.type);     // active | canceled | past_due
await setPlan(userId, plan, { status, source: "revenuecat", raw: e });
```

- **Idempotent:** key on `e.id`; ignore replays.
- On `EXPIRATION` / `CANCELLATION` (after period end) → set plan back to `free`.
- On `BILLING_ISSUE` → keep plan, set `status: "past_due"` (grace; see
  [../edge-cases.md](../edge-cases.md)).
- Also expose `GET /billing/sync` that calls the RC REST API to reconcile on
  demand (e.g. after a web return) so the UI updates without waiting on the webhook.

See [../data-model.md](../data-model.md) for the `setPlan` field writes and
[../api.md](../api.md) for the full endpoint list.

## Env / secrets (per stage, via SST)

```
RC_PUBLIC_KEY_IOS, RC_PUBLIC_KEY_ANDROID   # shipped in app build
RC_WEB_BILLING_PUBLIC_KEY                  # shipped in landing
RC_SECRET_API_KEY                          # Api only (REST reconcile)
RC_WEBHOOK_SECRET                          # Api only (verify webhook)
```
