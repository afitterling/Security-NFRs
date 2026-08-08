# Dynamic payment — variable & usage-based billing (additive)

> Status: **design sketch**. This spec is **additive**: the config-driven fixed
> tiers in [plans/plan-model.md](plans/plan-model.md) are kept exactly as they
> are. Dynamic payment is a layer *on top* — it never replaces the tier model.

The fixed catalog (Free/Starter/Pro/Enterprise) answers "pick a plan". Some
customers don't fit a fixed box: they want to pay only for what they use, buy a
one-off burst, or negotiate bespoke limits. "Dynamic payment" defines the model
that allows those **variable** charges without forking the plan system.

## Three mechanisms (compose freely)

| Mechanism | What the user gets | Billing shape |
|---|---|---|
| **A. Usage-based overage** | Go past a tier's daily cap instead of being hard-blocked | Metered, billed per extra unit (per call / per MB), in arrears |
| **B. Prepaid credits / top-ups** | Buy a bundle of extra calls/data on demand | One-time charge, balance consumed before the cap |
| **C. Per-customer custom limits** | Negotiated/bespoke limits (dynamic "enterprise") | Subscription whose limits come from metadata, not the catalog |

A fourth, smaller idea — **variable-amount checkout** (pay-what-you-want / custom
quote) — is just a checkout option that maps a paid amount → an entitlement; see
the bottom of this doc.

## Core principle — keep the tier as the *base*, make the delta dynamic

`userPlan()` already resolves a base `Plan` live from config. Dynamic payment
makes the **effective** limits = `base tier` ± **per-user dynamic state**:

```
effectiveLimits(user) = applyOverrides(planFor(user.plan), user.planOverrides)
decisionAtCap(user)   = credits? consume : overageEnabled && underSpendCap? meter+allow : block
```

The limiter stays the single gate. Dynamic payment only changes (1) the *limits*
it reads and (2) the *decision* it makes at the boundary.

## Data model additions (one user row, additive)

```ts
{
  // ...existing plan fields (plan, planStatus, ...)...

  // A. Usage-based overage
  overageEnabled?: boolean,        // user opted into pay-as-you-go past the cap
  overageCapMinor?: number,        // hard spend ceiling per period (anti bill-shock), minor units
  overageRate?: {                  // price per extra unit (else inherit from plan/provider)
    perCall?: number, perMB?: number, currency: "EUR",
  },
  overageUsedMinor?: number,       // accrued, uninvoiced spend this period (reset on renewal)
  overageMeterId?: string,         // provider meter / subscription item to report usage to

  // B. Prepaid credits
  credits?: { calls?: number, bytes?: number },  // remaining prepaid units

  // C. Per-customer custom limits (dynamic enterprise)
  planOverrides?: Partial<{        // overrides specific Plan fields for THIS user
    dailyCalls: number, dailyBytes: number,
    maxWebhooks: number, maxDevices: number, retentionDays: number,
  }>,
}
```

- Defaults absent ⇒ behave exactly like today (hard tiers, no overage, no credits).
- `planOverrides` is merged over the base plan in one place (`userPlan()`), so
  enforcement, `GET /me/plan`, and `GET /me/usage` all see the custom numbers.
- An optional `BillingEvents` / `UsageLedger` table is recommended once metering
  is on (idempotent unit accounting + audit). See [data-model.md](data-model.md).

## A. Usage-based overage (pay-as-you-go)

At the daily cap, instead of an immediate 402:

```
if (over daily cap) {
  if (!overageEnabled)                         -> block 402 quota_exceeded (today's behavior)
  if (overageUsedMinor >= overageCapMinor)     -> block 402 overage_cap_reached
  else { record 1 billable unit; allow }        // meter to provider, accrue overageUsedMinor
}
```

- **Metering rail:** RevenueCat does not natively meter usage, so overage runs on
  a usage-capable provider — **Stripe metered billing** or **Polar/Lemon Squeezy
  usage-based** (the hybrid rail from [checkout/overview.md](checkout/overview.md)).
  Report units to the provider's meter (Stripe `meterEvents` / subscription item
  usage records; Polar usage ingestion). The provider invoices in arrears.
- **Idempotent metering:** key each reported unit on `userId#YYYY-MM-DD#seq` so a
  retry never double-bills.
- **Bill-shock guard (required):** `overageCapMinor` is a hard ceiling; over it,
  block. Surface the live accrued overage in `GET /me/usage` and the dashboard.
- **Opt-in only:** overage is off by default — a user must explicitly enable it
  (so the default experience is still "predictable, never surprised").

## B. Prepaid credits / top-ups

A one-time purchase (not a subscription) adds to `credits`. The limiter consumes
credits **before** counting against the daily cap (or to extend it for the day):

```
consume(): credits.calls > 0 ? credits.calls-- (allow) : countAgainstDailyCap()
```

- Works with **any** provider, including a RevenueCat one-time (non-renewing)
  product or a Stripe/Polar one-off checkout.
- Credits never expire silently without disclosure; show the balance in the UI.
- Good for "I need a burst today" without committing to a higher tier.

## C. Per-customer custom limits (dynamic enterprise)

`planOverrides` lets sales close a bespoke deal — e.g. "Pro, but 5,000 calls/day
and 500 webhooks" — **without** a code change or a new catalog tier:

- The subscription stays a normal tier (e.g. `enterprise`); the custom numbers
  ride in provider metadata (RevenueCat subscriber attributes / Stripe price or
  subscription `metadata`).
- The webhook's `PlanChange` carries an optional `limits` patch → `setPlan` writes
  `planOverrides`.
- `userPlan()` merges overrides over the base tier. Enforcement and display both
  reflect the custom limits automatically.

## Variable-amount checkout (optional)

For "pay-what-you-want" or a custom quote, the checkout collects a **variable
amount** (Stripe custom-amount / Polar pay-what-you-want) and maps the paid
amount → an entitlement or a credit bundle via line-item metadata. The resulting
event still normalizes to a `PlanChange` (or a credit grant) → `setPlan`.

## API surface (additions to [api.md](api.md))

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/billing/overage` | `{ enabled, capMinor }` — opt in/out of pay-as-you-go + set the spend ceiling |
| `POST` | `/billing/topup` | `{ bundle }` — start a one-time checkout for a credit bundle → `{ url }` |
| `GET` | `/me/usage` | extend: also returns `credits`, `overage:{enabled,used,cap}` for the UI |

Provider webhooks emit usage-invoice / one-time-payment events that credit the
ledger; all converge on the existing `setPlan` / a `grantCredits` writer.

## Enforcement changes (see [enforcement/rate-limiting.md](enforcement/rate-limiting.md))

- The boundary decision changes from "block at cap" to the
  credits → overage → block ladder above — **only for opted-in users**; everyone
  else keeps today's hard block.
- Burst guard and sustained-overage auto-disable stay (flood/cost defense), but
  the auto-disable threshold must account for legitimate paid overage.

## Edge cases

- **Bill-shock:** `overageCapMinor` is mandatory when `overageEnabled`; never meter
  past it. Show accrued spend live.
- **Downgrade with credits:** prepaid credits survive a downgrade (already paid);
  `planOverrides` are cleared when the custom deal ends.
- **Provider split:** overage/metering uses the hybrid rail even if the base
  subscription is RevenueCat — make `setPlan`/`grantCredits` treat the metering
  provider as authoritative for *usage*, RC for *entitlement*.
- **Idempotency:** unit metering and credit grants are idempotent on a stable id,
  same discipline as billing webhooks.

## Cross-reference

- Base model (unchanged): [plans/plan-model.md](plans/plan-model.md) ·
  [plans/limits.md](plans/limits.md)
- Providers that can meter / take one-offs: [checkout/comparison.md](checkout/comparison.md) ·
  [checkout/stripe.md](checkout/stripe.md) · [checkout/polar.md](checkout/polar.md)
- Data + API: [data-model.md](data-model.md) · [api.md](api.md)
