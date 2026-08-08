# Provider comparison & decision guide

## Fees (approximate, EU, consumer subscriptions)

| Provider | Headline fee | Merchant of Record? (VAT handled) | Works on | Via RevenueCat? |
|---|---|---|---|---|
| **Apple App Store** | 15% (Small Business) / 30% | Yes (Apple) | iOS | ✅ |
| **Google Play** | 15% / 30% | Yes (Google) | Android | ✅ |
| **Stripe (RC Web Billing)** | ~1.5% + €0.25 (EU cards) + RC fee | No (Stripe Tax extra) | Web | ✅ (recommended web) |
| **Stripe (direct)** | ~1.5% + €0.25 | No | Web | ❌ (hybrid) |
| **PayPal** | ~2.9–3.4% + fixed | No | Web | ❌ (hybrid) |
| **Polar.sh** | ~4% + €0.40 | **Yes** | Web | ❌ (hybrid) |
| **Lemon Squeezy / Paddle** | ~5% + €0.50 | **Yes** | Web | ❌ (hybrid) |

RevenueCat itself: free under a monthly tracked-revenue threshold, then ~1% of
tracked revenue. It does **not** add fees on top of the App Store / Play cut.

## The trade-off in one line

- **Lowest fee, most work, tax-on-you:** Stripe direct.
- **Lowest fee on web + unified mobile, minimal work:** RevenueCat (Stripe under
  the hood) — **recommended**.
- **Cheapest *net* for EU consumers (tax handled), more work:** Polar.sh (MoR).
- **PayPal:** only for audience demand; most complex to reconcile.

## Hard constraints (not preferences)

- **iOS digital subscriptions MUST use Apple IAP.** You cannot sell the
  subscription with Stripe/PayPal *inside* the iOS app. Web checkout is for the
  website only. → mobile = Apple/Google, full stop.
- Therefore you need **at least two** billing rails (mobile IAP + web). RevenueCat
  exists precisely to unify them into one entitlement.

## Recommendation

1. **Phase 1 (ship subscriptions):** RevenueCat everywhere — Apple + Google +
   RC Web Billing (Stripe). One SDK family, one webhook, one `user.plan`. This
   alone satisfies "select a plan on login + checkout on app and web."
2. **Phase 2 (if needed):** add **Polar.sh** as a web provider (cheaper-net EU,
   MoR tax) and/or **PayPal** for demand — via the `BillingProvider` abstraction,
   no change to enforcement.

## Cross-reference

- Architecture: [overview.md](overview.md) · RevenueCat detail: [revenuecat.md](revenuecat.md)
- Mobile store setup: [apple-google-iap.md](apple-google-iap.md)
- Hybrid adapters: [stripe.md](stripe.md) · [paypal.md](paypal.md) · [polar.md](polar.md)
