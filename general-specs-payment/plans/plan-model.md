# Plan model — config-driven

Plans are **data**, defined in one place and consumed everywhere (enforcement,
UI, checkout). This is the "customized per app" requirement: editing the catalog
re-tiers the whole system without code changes elsewhere.

## Source of truth

Today: `sst/src/lib/plans.ts` already defines the catalog and is read live at
ingest by `ratelimit.ts`. We **extend** that file rather than introducing a new
system.

Current shape:

```ts
export type PlanId = "free" | "starter" | "pro" | "enterprise";
export interface Plan {
  id: PlanId;
  name: string;
  dailyCalls: number;
  dailyBytes: number;
  priceLabel: string;   // <- to be replaced by structured pricing
}
```

## Proposed shape

```ts
export type PlanId = "free" | "starter" | "pro" | "enterprise";

export interface PlanPrice {
  monthly: number;        // minor units? -> keep major units + currency for clarity
  annual: number;         // ≈ monthly * 10 (2 months free)
  currency: "EUR";        // canonical; localized display handled by i18n
  // External provider price/product ids, filled per environment:
  stripe?: { monthly: string; annual: string };
  paypal?: { monthly: string; annual: string };
  polar?:  { monthly: string; annual: string };
  apple?:  { monthly: string; annual: string };  // StoreKit product ids
  google?: { monthly: string; annual: string };  // Play base plan ids
}

export interface Plan {
  id: PlanId;
  name: string;
  // Limits — the single source the limiter enforces (see plans/limits.md):
  dailyCalls: number;
  dailyBytes: number;
  maxWebhooks: number;    // NEW — enforce at POST /webhooks
  maxDevices: number;     // NEW — enforce at device register
  retentionDays: number;  // NEW — message TTL per plan
  // Pricing (free tier has zero prices, no provider ids):
  price: PlanPrice;
  // Marketing copy keys resolved through i18n (not literal strings here):
  featuresKey: string;
}
```

## Rules

- **`free` is the default.** New / unsubscribed / lapsed users resolve to `free`
  via `planFor()` (already implemented, falls back safely on unknown ids).
- **Plan is read live**, never cached on the webhook/message — instant up/downgrade.
- **Prices are canonical in EUR**; the landing site localizes display via the
  existing `i18n.ts` machinery. Provider price ids are environment config
  (test vs prod), not committed secrets.
- Adding a tier = add a `PlanId` + an entry. Enforcement, picker, and checkout
  pick it up automatically.

See [catalog.md](catalog.md) for concrete values and [limits.md](limits.md) for
the enforced limit matrix.
