# WebhookPush — Payment & Subscription Spec

A subscription-based billing model for WebhookPush: after login the user picks a
plan, sees transparent pricing/limits up front, checks out via one of several
providers, and the app **enforces the plan's limits automatically**.

> Status: **design sketch**. No payment SDK is wired yet. The quota *enforcement*
> engine already exists in code (see [enforcement/rate-limiting.md](enforcement/rate-limiting.md));
> the missing half is persisting the chosen plan onto the user. This spec defines
> both halves.

## Folder index

| Area | Doc |
|---|---|
| Plan model (config-driven) | [plans/plan-model.md](plans/plan-model.md) |
| Dynamic payment (usage-based / credits / custom — additive) | [dynamic-payment.md](dynamic-payment.md) |
| Plan catalog + prices | [plans/catalog.md](plans/catalog.md) |
| Canonical limits (what gets enforced) | [plans/limits.md](plans/limits.md) |
| Checkout abstraction | [checkout/overview.md](checkout/overview.md) |
| Stripe adapter | [checkout/stripe.md](checkout/stripe.md) |
| PayPal adapter | [checkout/paypal.md](checkout/paypal.md) |
| Polar.sh (cheap MoR) adapter | [checkout/polar.md](checkout/polar.md) |
| Apple StoreKit + Google Play | [checkout/apple-google-iap.md](checkout/apple-google-iap.md) |
| Provider comparison (fees/rules) | [checkout/comparison.md](checkout/comparison.md) |
| Data model changes | [data-model.md](data-model.md) |
| Billing API endpoints | [api.md](api.md) |
| Rate limiting / quota enforcement | [enforcement/rate-limiting.md](enforcement/rate-limiting.md) |
| Plan-selection UI | [ui/plan-selection.md](ui/plan-selection.md) |
| Usage + billing UI (transparency) | [ui/usage-and-billing.md](ui/usage-and-billing.md) |
| Restore Purchases (subscription restore) | [ui/restore-purchases.md](ui/restore-purchases.md) |
| Edge cases | [edge-cases.md](edge-cases.md) |

## Decisions (locked with the user)

- **Plan model:** config-driven tiers — plans are *data* (extend `sst/src/lib/plans.ts`),
  not hardcoded UI, so they can be customized per deployment.
- **Billing periods:** monthly **and** annual (annual ≈ 2 months free).
- **Providers:** **RevenueCat is the unifying billing layer** across mobile + web
  (one entitlement → `user.plan`, one webhook to the Api). Under it:
  - **Apple StoreKit 2 + Google Play Billing** — the mobile app, via
    `react-native-purchases` (App Store rules mandate IAP for digital subs).
  - **RevenueCat Web Billing** — the web (SST/Remix landing) via
    `@revenuecat/purchases-js`; Stripe-backed cards.
  - A **provider abstraction** is still defined so **PayPal** and a **cheap
    merchant-of-record (Polar.sh / Lemon Squeezy / Paddle)** can be added on web
    *outside* RevenueCat if desired — RC Web Billing is Stripe-only.
  See [checkout/overview.md](checkout/overview.md) and [checkout/revenuecat.md](checkout/revenuecat.md).
- **Enforcement:** reuse the existing limiter; the subscription's job is to set
  `user.plan`. Over-limit ingest is **blocked** (HTTP 402/429) and sustained
  overage auto-disables the webhook (already implemented).
- **Transparency:** the dashboard reflects the live plan + usage-vs-limit; the
  plan picker shows localized prices and exactly what each tier includes.

## Open items the user must confirm

1. **Final prices.** `plans.ts priceLabel` (€4/€10/€100) and the landing
   `i18n.ts PLAN_PRICES` (€9.99/€17/€89) **disagree** today. This spec proposes
   one canonical table (see [plans/catalog.md](plans/catalog.md)) — confirm the numbers.
2. **MoR provider choice** (Polar vs Lemon Squeezy vs Paddle).
3. **Mobile pricing** — IAP prices must be set in App Store / Play Console and
   typically differ from web (stores take 15–30%). See [checkout/comparison.md](checkout/comparison.md).
4. Where this spec should ultimately live (currently a sibling of the `webhook` repo).
