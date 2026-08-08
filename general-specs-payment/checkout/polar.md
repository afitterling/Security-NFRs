# Polar.sh — the "cheap" merchant-of-record (hybrid / direct only)

The low-fee web option. Like PayPal, it sits **outside** RevenueCat (RC Web
Billing is Stripe-only), so it's part of the hybrid path. Its appeal: it's a
**merchant of record (MoR)** — it handles **EU VAT** and remits tax for you, which
for EU consumer sales is often cheaper *net* than Stripe-direct (where VAT/MOSS is
your problem).

## Adapter

- Create Products + recurring Prices in Polar; store ids in `plans.ts` `price.polar`.
- `POST /billing/checkout` (provider=polar) → create a **Checkout** via the Polar
  API → return the hosted checkout `url`.
- Customer portal: Polar's hosted customer portal link for manage/cancel.
- Webhook `POST /billing/polar/webhook`: verify the `webhook-signature` (HMAC),
  handle `subscription.active|updated|canceled|revoked` + `order.*` → emit
  `PlanChange`. Map Polar product/price → `PlanId`.

## Drop-in alternatives (same MoR role)

| MoR | Fee (approx) | Notes |
|---|---|---|
| **Polar.sh** | ~4% + 40¢ | developer-first, cheapest MoR, open-source |
| **Lemon Squeezy** | ~5% + 50¢ | now a Stripe company; mature |
| **Paddle** | ~5% + 50¢ | enterprise-grade tax handling |

All three expose: hosted checkout + customer portal + signed webhooks → the same
`BillingProvider` adapter shape. Picking one is a config swap, not an architecture
change.

## When to use vs RevenueCat-only

- Want the **simplest** stack and don't need PayPal/MoR-tax on web →
  RevenueCat-only (Stripe under the hood); skip Polar.
- Selling to **EU consumers at volume** and want tax off your plate, or want a
  cheaper-net web option than Stripe-direct → add Polar as the web provider while
  RevenueCat keeps mobile.
