# Billing API (SST Api — Hono)

New endpoints on the existing Hono app in `sst/src/api.ts`. Auth is the existing
`Authorization: Bearer <token>` (or legacy `x-user-id`) unless noted. Webhooks are
unauthenticated HTTP but **verified by provider signature/secret**.

## Customer-facing

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/plans` | none | Public catalog (ids, names, limits, EUR prices). Drives picker + landing. |
| `GET` | `/me/plan` | user | Current `{ plan, planStatus, planRenewsAt, planPeriod }`. |
| `GET` | `/me/usage` | user | `{ plan, limits, used }` for today (calls, bytes, webhooks, devices). Powers usage bars. |
| `POST` | `/billing/checkout` | user | Body `{ plan, period }`. Returns `{ url }` for the active web provider (RevenueCat Web Billing by default; Stripe/PayPal/Polar in hybrid). |
| `POST` | `/billing/portal` | user | Returns `{ url }` to manage/cancel (RC customer portal or Stripe portal). |
| `POST` | `/billing/sync` | user | On-demand reconcile via RC REST API; returns refreshed `/me/plan`. Used on web return + app foreground. |

> Mobile does **not** call `/billing/checkout` — purchases happen in-app through
> `react-native-purchases`; the webhook updates the plan. The app calls
> `/me/plan` + `/me/usage` for display and `/billing/sync` after a purchase.

## Webhooks (provider → Api)

| Method | Path | Verify | Emits |
|---|---|---|---|
| `POST` | `/billing/revenuecat/webhook` | `Authorization` == `RC_WEBHOOK_SECRET` | `PlanChange` → `setPlan` |
| `POST` | `/billing/stripe/webhook` | `Stripe-Signature` (hybrid only) | `PlanChange` |
| `POST` | `/billing/paypal/webhook` | PayPal webhook id verify (hybrid only) | `PlanChange` |
| `POST` | `/billing/polar/webhook` | `webhook-signature` HMAC (hybrid only) | `PlanChange` |

All webhooks: **idempotent** on event id (see [data-model.md](data-model.md)
BillingEvents), always return 200 quickly after persisting, do work async if needed.

## Shared internals

```ts
function entitlementToPlan(active: string[]): PlanId; // highest active → plan, else "free"
function mapStatus(eventType: string): "active" | "canceled" | "past_due";
async function setPlan(userId, plan, meta): Promise<void>;  // idempotent UpdateItem
```

## Enforcement touch-points (existing endpoints, changed)

- `POST /webhooks` — add `maxWebhooks` check (count via `ByUser` GSI) → 402 over cap.
- device register — add `maxDevices` check → 402 over cap.
- ingest — already gated; standardize the **402 quota_exceeded** body
  ([plans/limits.md](plans/limits.md)) and include `upgradeUrl`.

## Landing routes (Remix, `vite.config.ts`)

Add manual routes: `/billing` (picker), `/billing/return` (post-checkout sync),
plus a `/plans`-backed pricing section. Resource routes follow the existing
manual-route convention (same pattern as the favicon/robots routes).
