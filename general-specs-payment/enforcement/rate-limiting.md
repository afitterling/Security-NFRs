# Rate limiting & quota enforcement

## TL;DR — it already exists

`sst/src/lib/ratelimit.ts` is a working, plan-aware limiter that **blocks** at
ingest. The user's ask ("build a rate limiter that blocks users when they reach
these levels") is ~90% done in code. The missing piece is **setting `user.plan`**
so the limiter enforces the *paid* tier instead of always `free`.

## What exists today

| Function | Role |
|---|---|
| `userPlan(userId)` | reads `user.plan` **live** from Users; `planFor()` → `free` fallback |
| `consumeUsage(userId, bytes)` | **the main gate** — one `UpdateItem` counts call+bytes, returns `callsAllowed` / `dataAllowed` / `autoDisable` against the live plan |
| `consumeQuota` / `consumeData` | call-only / data-only variants |
| `withinBurst(webhookId)` | 20 req / 10s hard cap (plan-independent) |
| `tooManyAuthAttempts(bucket,…)` | login/signup brute-force throttle |

Ingest order (from the file's own header): unknown/disabled webhook → bad secret
→ burst guard → daily quota (live plan) → sustained overage auto-disable. Counters
live in the `Usage` table keyed `userId#YYYY-MM-DD`, TTL 2 days.

## The gap

```ts
// userPlan() reads user?.plan — but NOTHING ever writes user.plan.
// => planFor(undefined) => free for everyone, forever.
```

## Changes to make

1. **Persist the plan (highest value).** Add `setPlan(userId, plan, meta)` (see
   [../data-model.md](../data-model.md)); call it from the RevenueCat webhook
   (and any direct provider webhook). After this, the limiter enforces real tiers.

2. **Extend limits beyond daily counts** ([../plans/limits.md](../plans/limits.md)):
   - `maxWebhooks` — at `POST /webhooks`, count user's webhooks (the `ByUser`
     GSI) and reject create over cap.
   - `maxDevices` — at device register, count devices and reject over cap.
   - `retentionDays` — set `Messages.expiresAt = now + plan.retentionDays` instead
     of the flat 30 days.

3. **Make the block response explicit.** Return the structured 402 body from
   [../plans/limits.md](../plans/limits.md) (metric, used, limit, plan, upgradeUrl)
   so the app/dashboard can explain it and deep-link to `/billing`.

4. **Reflect usage.** Add `GET /me/usage` returning `{ plan, limits, used }`
   (read today's `Usage` row + `planFor`). Powers the transparency UI.

## Deliberately unchanged

- Burst cap stays plan-independent (flood/cost defense, not a sales lever).
- Auto-disable at 5× daily quota stays — it's the cost backstop on the public
  Function URL. Edge/WAF throttling (sst.config TODO) remains the real flood wall.
- Counting model (single merged `UpdateItem`) stays — it's a cost optimization.

## Test plan

- Unit: `planFor` mapping, `consumeUsage` boundary (used == limit allowed, +1
  blocked), `maxWebhooks`/`maxDevices` at cap.
- Integration: set `user.plan=pro`, replay >100 ingest calls, assert 402 at the
  right count; downgrade to `free`, assert immediate tighter cap (live read).
- Idempotency: replay a RevenueCat webhook event id, assert single plan write.
