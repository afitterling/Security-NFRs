# Stripe

In the **recommended** architecture, Stripe is used **through RevenueCat Web
Billing** — you don't integrate the Stripe SDK directly; RevenueCat manages the
Stripe account, checkout, and portal. Configure the Stripe connection in the
RevenueCat dashboard and you're done on web.

## Direct Stripe (hybrid only)

Use this **only** if you want web checkout outside RevenueCat (e.g. to also offer
PayPal/Polar and keep a single Stripe relationship). Then implement the
`BillingProvider` adapter:

- `stripe.checkout.sessions.create({ mode: "subscription", line_items: [priceId], ... })`
  → return `session.url` from `POST /billing/checkout`.
- Prices: one Stripe **Price** per tier×period; ids stored in `plans.ts` `price.stripe`.
- Customer portal: `stripe.billingPortal.sessions.create()` for `POST /billing/portal`.
- Webhook `POST /billing/stripe/webhook`, verify `Stripe-Signature`, handle
  `customer.subscription.{created,updated,deleted}` + `invoice.payment_failed`
  → emit `PlanChange`. Map `price.id` → `PlanId`.
- Store `stripeCustomerId` on the user for portal + reconcile.

## Notes

- Fees: ~1.5% (EU cards) + €0.25, **plus you owe EU VAT/MOSS yourself** (Stripe
  Tax is extra). This is why a merchant-of-record (Polar) may be cheaper net for
  EU consumer sales despite a higher headline %. See [comparison.md](comparison.md).
- Running Stripe both directly *and* via RevenueCat on the same account is
  possible but confusing — pick one path for web. Recommended: RC Web Billing.
