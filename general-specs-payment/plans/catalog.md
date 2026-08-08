# Plan catalog (proposed)

> **Prices are a proposal — confirm before wiring.** Today `plans.ts priceLabel`
> (€0/€4/€10/€100) disagrees with landing `i18n.ts PLAN_PRICES` (€0/€9.99/€17/€89).
> This table proposes the **landing numbers** as canonical (they're already
> localized and customer-facing) and adds annual ≈ 10× monthly (2 months free).

| Plan | Monthly | Annual | Daily calls | Daily data | Max webhooks | Max devices | Retention |
|---|---|---|---|---|---|---|---|
| **Free** | €0 | €0 | 10 | 1 MB | 1 | 1 | 7 days¹ |
| **Starter** | €9.99 | €99 | 100 | 20 MB | 5 | 3 | 30 days |
| **Pro** | €17 | €170 | 1,000 | 250 MB | 25 | 10 | 90 days |
| **Enterprise** | €89 | €890 | 10,000 | 5 GB | 200 | 50 | 365 days |

¹ Free webhooks auto-disable after 7 days unused (existing `expire.ts`); message
TTL is currently a flat 30 days for everyone (`Messages.expiresAt`) — per-plan
retention is a **new** limit this spec introduces.

## Notes

- **Daily calls / daily data already exist** in `plans.ts` and are enforced.
  Only the *values* and the new columns (webhooks/devices/retention) are added.
- **Currency:** canonical EUR; the landing localizes display via `i18n.ts`
  (`PLAN_PRICES` already has USD/CNY/JPY/PLN/SEK… variants). Keep that, but read
  the *limits* from `plans.ts` so marketing copy and enforcement can't drift.
- **Store pricing differs.** Apple/Google take 15–30%, and store price tiers are
  fixed points — IAP prices are set in App Store Connect / Play Console and will
  not be exactly €9.99/€17/€89. RevenueCat reports the store price for display;
  do not hardcode mobile prices. See [../checkout/comparison.md](../checkout/comparison.md).
- **Annual discount** rendered as "2 months free" in the picker for transparency.

## Free-tier intent

Free is a real, permanently-usable tier (10 calls/day, 1 webhook) — "No card, no
setup" matches existing landing copy. It exists to convert, not to trial-and-expire.
