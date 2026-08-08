# UI — plan selection (post-login)

Goal: after login the user picks a plan with **transparent pricing and limits up
front** — "planned transparency shown up right."

## Web (`sst/landing`, Remix) — `/billing`

Layout: a responsive pricing grid, one card per tier (Free / Starter / Pro /
Enterprise), reusing the dark + electric-lime brand.

Each card shows, with **no hidden surprises**:
- Price: monthly **and** annual toggle, annual labeled "2 months free".
- The concrete limits from `GET /plans`: daily calls, daily data, max webhooks,
  max devices, retention — the *same numbers the limiter enforces*.
- The user's **current** plan badged ("Your plan"); upgrade/downgrade CTA on others.
- Localized currency via existing `i18n.ts` (`PLAN_PRICES`).

Flow:
1. Card CTA → `POST /billing/checkout { plan, period }` → redirect to `{ url }`
   (RevenueCat Web Billing / Stripe).
2. Return to `/billing/return` → call `POST /billing/sync` → show new plan + toast.
3. Manage/cancel → `POST /billing/portal` → provider portal.

Monthly/annual toggle updates prices live; the included limits don't change with
period (only price does) — keep that obvious.

## Mobile (`webhook-app`, Expo) — Plans screen

- Fetch `Purchases.getOfferings()` and render packages. **Prices come from the
  store** (localized, tax-inclusive) — never hardcode.
- Buy → `Purchases.purchasePackage()` → on success `POST /billing/sync` then
  refresh `/me/plan`.
- **Restore purchases** button (App Store requirement).
- Show the same limit list as web so the value is identical across platforms.

## Transparency rules

- Always show what the user gets **and** what they're currently using (link to the
  usage view, [usage-and-billing.md](usage-and-billing.md)).
- Never advertise a number the limiter doesn't enforce — `/plans` is generated
  from `plans.ts`, the same source enforcement reads.
- Free is presented as a permanent tier, not a trial.
