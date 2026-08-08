# Edge cases & policies

## Upgrade / downgrade

- **Upgrade:** effective immediately. The next ingest reads the higher `dailyCalls`
  (live read — no cache). Provider handles proration (RevenueCat/Stripe).
- **Downgrade:** apply at period end (avoid refund math). Keep the higher plan
  until `planRenewsAt`, then `setPlan(free|lower)`. If the user is already *over*
  the lower plan's `maxWebhooks`/`maxDevices` at downgrade, **don't delete**
  anything — block *new* creates until they're back under cap (grandfather + soft-cap).

## Cancellation

- `CANCELLATION` event (user turned off auto-renew) → keep plan, set
  `planStatus = canceled`, show "active until <renewsAt>".
- `EXPIRATION` (period actually ended) → `setPlan(free)`.

## Billing issues / dunning

- `BILLING_ISSUE` → `planStatus = past_due`, **keep** the paid plan during the
  provider's grace/retry window. Show amber banner. Let the provider
  (RevenueCat/Stripe) run retries; on recovery → `active`, on final failure →
  `EXPIRATION` → `free`.

## Refunds / chargebacks

- Provider refund event → `setPlan(free)` (or prior tier) immediately; log to
  BillingEvents. Apple/Google refunds arrive as RevenueCat events too.

## Idempotency & ordering

- Webhooks may arrive out of order or duplicated. Guard on event id
  (`lastBillingEventId` / BillingEvents). Prefer the event with the latest
  provider timestamp; when in doubt, call `/billing/sync` to read RC's current truth.

## Cross-platform conflicts (RevenueCat resolves, but know the rules)

- A user could subscribe on web (Stripe via RC) **and** iOS. RevenueCat reports
  one customer with possibly multiple active entitlements → resolve to the
  **highest** tier; surface a "you have two active subscriptions" note and point
  them to the relevant store to cancel the redundant one (you cannot cancel an
  Apple sub server-side).
- Plan changes initiated in-app go through the store; never try to "set plan"
  from your own UI for store-billed users — it must come from the webhook.

## App Store / Play policy

- iOS: digital subscriptions **must** use IAP — do not link out to web checkout
  from inside the iOS app to dodge fees (rejection risk). Web checkout is for the
  website/desktop only.
- Provide **Restore Purchases** (Apple requirement).
- Show price, period, and auto-renew terms before purchase (store requirement).

## Free-tier lifecycle

- Free webhooks auto-disable after 7 days unused (`expire.ts`) — unchanged.
- A lapsed paid user falls to `free`; if over free caps, soft-cap as in Downgrade.

## Failure modes

- RevenueCat webhook down / delayed → enforcement still works on last-known
  `user.plan`; `/billing/sync` reconciles. Never block the auth flow on billing.
- Provider outage during checkout → user stays on current plan; nothing charged.
