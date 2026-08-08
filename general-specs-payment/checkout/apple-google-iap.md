# Apple StoreKit + Google Play (via RevenueCat)

RevenueCat abstracts the runtime, but you still configure the stores. This is
what's required at the store level regardless of RevenueCat.

## Apple (App Store Connect)

- Create **auto-renewable subscriptions** in a Subscription **Group** (so
  upgrade/downgrade/crossgrade between tiers is handled by Apple). One group;
  products `starter_monthly`, `starter_annual`, `pro_*`, `ent_*`.
- Set price tiers per territory (Apple's fixed price points — won't be exactly
  €9.99/€17/€89). Add localized display name/description.
- **App Store Server Notifications v2** → point to RevenueCat's URL (RC ingests
  them). RevenueCat does receipt validation; you don't handle StoreKit receipts.
- StoreKit **sandbox** testers for QA; `.storekit` config file for local testing.
- Requirements: **Restore Purchases**, show terms + auto-renew disclosure, link
  to manage subscriptions (`itms-apps://apps.apple.com/account/subscriptions`).

## Google (Play Console)

- Create **subscriptions** with monthly/annual **base plans** + offers.
- **Real-time developer notifications (RTDN)** via Pub/Sub → RevenueCat.
- License testers for QA.

## Expo specifics

- `react-native-purchases` needs a **dev build / prod build** (config plugin),
  not Expo Go. Add the plugin in `app.json` and rebuild.
- iOS: StoreKit capability; Android: BILLING permission (handled by the lib).
- `Purchases.logIn(userId)` ties the store customer to our `userId` so the
  webhook's `app_user_id` matches the Users key.

## What you do NOT build

- No receipt validation, no signature parsing, no renewal cron — RevenueCat
  normalizes all store events into the single `/billing/revenuecat/webhook`
  (see [revenuecat.md](revenuecat.md)).
