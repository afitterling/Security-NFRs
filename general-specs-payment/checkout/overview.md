# Checkout architecture — RevenueCat-centric

## The shape

```
  iOS app  ──┐                         ┌── App Store (StoreKit 2)
             │   react-native-purchases │
 Android app ┤───────────────▶ RevenueCat ──┤── Google Play Billing
             │                          └── RC Web Billing (Stripe)
  Web (Remix)┘   @revenuecat/purchases-js
                          │
                          │ webhook (subscription events)
                          ▼
                 SST Api  ──▶  sets user.plan  ──▶  existing limiter enforces
```

**One source of truth:** RevenueCat tracks the customer's active **entitlement**
(e.g. `pro`). Whatever platform they paid on, RevenueCat tells the Api the
current entitlement, and the Api writes `user.plan`. The limiter
([../enforcement/rate-limiting.md](../enforcement/rate-limiting.md)) already reads
`user.plan` live — so enforcement "just works" once the field is set.

## Mapping entitlements → plans

RevenueCat **entitlement id == `PlanId`**. Configure in the RevenueCat dashboard:

| RC Entitlement | `PlanId` | Offerings (products) |
|---|---|---|
| `starter` | `starter` | `starter_monthly`, `starter_annual` |
| `pro` | `pro` | `pro_monthly`, `pro_annual` |
| `enterprise` | `enterprise` | `ent_monthly`, `ent_annual` |
| *(none active)* | `free` | — |

The Api resolves the **highest active entitlement** to a plan; no active
entitlement → `free` (handled by `planFor()` fallback).

## Provider abstraction (for non-RevenueCat options)

Even with RevenueCat as the spine, define a thin server-side interface so PayPal
/ Polar can be added on web later without touching call sites:

```ts
interface BillingProvider {
  id: "revenuecat" | "stripe" | "paypal" | "polar";
  // Start a web checkout; returns a redirect/checkout URL or client token.
  createCheckout(userId: string, plan: PlanId, period: "monthly" | "annual"): Promise<{ url: string }>;
  // Verify + normalize an inbound provider webhook into a PlanChange.
  handleWebhook(req: Request): Promise<PlanChange | null>;
}

interface PlanChange { userId: string; plan: PlanId; status: "active" | "canceled" | "past_due"; source: string; }
```

- **Default:** only the `revenuecat` provider is registered.
- **Hybrid:** register `paypal` / `polar` for web; their webhooks also emit a
  `PlanChange`. RevenueCat remains authoritative for mobile.
- All providers converge on the same `PlanChange` → `setPlan(userId, plan)`.

## Why RevenueCat over wiring each store directly

- StoreKit 2 receipt validation, App Store Server Notifications v2, Google
  Play RTDN, renewals, grace periods, billing retry, refunds — all handled and
  normalized by RevenueCat into one webhook + one customer-info object.
- Cross-platform: a user who subscribes on iOS is recognized on web/Android.
- One SDK family, one set of test sandboxes to reason about.

Trade-offs and the PayPal/Polar caveat: see [comparison.md](comparison.md).
Detailed integration: [revenuecat.md](revenuecat.md).
