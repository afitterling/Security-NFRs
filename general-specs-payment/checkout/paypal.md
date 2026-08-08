# PayPal (hybrid / direct only)

RevenueCat Web Billing does **not** support PayPal. If PayPal is required, it is a
**direct web adapter** alongside RevenueCat (which still owns mobile). This is the
"hybrid" path — adds a second reconciliation source.

## Adapter

- Use **PayPal Subscriptions**: create a Product + Billing **Plans** (one per
  tier×period) in the PayPal dashboard or API; store plan ids in `plans.ts`
  `price.paypal`.
- `POST /billing/checkout` (provider=paypal) → create subscription →
  return the approval `links[].href` (rel `approve`) as `{ url }`.
- Return URL → `/billing/return` → `POST /billing/sync` verifies subscription
  status via the PayPal API.
- Webhook `POST /billing/paypal/webhook`: subscribe to
  `BILLING.SUBSCRIPTION.{ACTIVATED,CANCELLED,SUSPENDED,EXPIRED}` and
  `PAYMENT.SALE.COMPLETED`; **verify** via PayPal webhook-id verification; emit
  `PlanChange`.

## Cost / caveats

- Fees ~2.9–3.4% + fixed; **not** a merchant of record — VAT is on you.
- No unified entitlement with RevenueCat: a PayPal-billed user's `user.plan` is
  set by the PayPal webhook directly. Make sure `entitlementToPlan` / `setPlan`
  treat `source: "paypal"` as authoritative for that user and don't get clobbered
  by an empty RevenueCat sync.
- Recommend offering PayPal **only** if there's clear demand; it materially
  increases billing complexity (two truths to reconcile).
