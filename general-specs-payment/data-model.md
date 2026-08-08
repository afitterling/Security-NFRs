# Data model changes

Minimal, additive changes to the existing DynamoDB tables (defined in
`sst/sst.config.ts`). No new table is strictly required; an optional
`BillingEvents` table helps with idempotency/audit.

## Users table — add subscription fields

Current `Users` item is keyed by `userId` (the email) and holds auth/identity
fields only. Add:

```ts
{
  // ...existing...
  plan: PlanId,                  // "free" | "starter" | "pro" | "enterprise"
  planStatus: "active" | "past_due" | "canceled",
  planSince: number,            // epoch ms when current plan took effect
  planRenewsAt?: number,        // next renewal (from provider), for display
  planPeriod?: "monthly" | "annual",
  planSource?: "revenuecat" | "stripe" | "paypal" | "polar",

  // Provider customer/subscription handles (for reconcile + portal links):
  rcAppUserId?: string,         // == userId; explicit for clarity
  stripeCustomerId?: string,    // if direct Stripe used (hybrid)
  // (RevenueCat is the authority; these are conveniences.)

  lastBillingEventId?: string,  // idempotency guard for webhooks
  lastBillingEventAt?: number,
}
```

- **Default:** absent `plan` ⇒ `free` (no migration/backfill needed — `planFor()`
  already handles `undefined`).
- `setPlan(userId, plan, { status, period, renewsAt, source, eventId })` writes
  these via a single `UpdateItem`, guarded by `eventId != lastBillingEventId`.

## Messages table — per-plan retention

Today `expiresAt = now + 30d` (flat). Change the writer (`ingest.ts`) to:

```ts
const plan = await userPlan(userId);
expiresAt = nowSec + plan.retentionDays * 86400;
```

DynamoDB TTL on `expiresAt` then expires per plan automatically. (Optionally also
filter reads to `retentionDays` so a downgrade hides older items immediately.)

## Usage table — unchanged

Keys already support per-user/day counters (`userId#YYYY-MM-DD`) and per-webhook
stats. `GET /me/usage` reads today's row; no schema change.

## Optional: BillingEvents table (recommended for audit/idempotency)

```
BillingEvents: { pk: eventId (string) }  TTL: expiresAt (e.g. 90 days)
  provider, type, userId, plan, raw, receivedAt
```

- Write-once on webhook receipt; if `eventId` already present, skip (idempotent).
- Cheaper/cleaner than overloading the user row if event volume grows.
- Doubles as a billing audit log for support.

## Plan config — `plans.ts`

Extend `Plan` with `maxWebhooks`, `maxDevices`, `retentionDays`, and structured
`price` (see [plans/plan-model.md](plans/plan-model.md)). This stays the single
source of truth read by both enforcement and the UI.
