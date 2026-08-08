# Canonical limits — what the system enforces

This is the contract between billing and enforcement. Each limit names **where**
it is checked and **what happens** on breach. Values live in `plans.ts` (see
[catalog.md](catalog.md)); this doc is the enforcement semantics.

| Limit | Field | Enforced at | On breach |
|---|---|---|---|
| Daily calls | `dailyCalls` | ingest (`consumeUsage`) | **block** request → 402/429; auto-disable webhook at 5× (existing) |
| Daily data | `dailyBytes` | ingest (`consumeUsage`) | **block** request → 402/429 |
| Burst | hardcoded 20/10s | ingest (`withinBurst`) | **block** → 429 (plan-independent flood guard) |
| Max webhooks | `maxWebhooks` *(new)* | `POST /webhooks` | **reject create** → 402 with upgrade hint |
| Max devices | `maxDevices` *(new)* | device register | **reject register** → 402 with upgrade hint |
| Retention | `retentionDays` *(new)* | message write (TTL) + read filter | older messages expire via DynamoDB TTL |

## Status today vs. to build

- ✅ **Daily calls / data / burst / auto-disable** — fully implemented in
  `sst/src/lib/ratelimit.ts`; read the live plan via `userPlan()`.
- ❌ **`user.plan` is never written** — so every user is effectively `free`.
  This is the single highest-value fix (see [../enforcement/rate-limiting.md](../enforcement/rate-limiting.md)).
- ❌ **maxWebhooks / maxDevices** — not yet enforced; add a count-check at create.
- ❌ **Per-plan retention** — `Messages.expiresAt` is a flat 30 days; make it
  `now + retentionDays`.

## Block semantics (the "blocks users when they reach these levels")

When over the **daily** limit, ingest returns a structured error so the app and
dashboard can show *exactly* why and what to do:

```http
HTTP/1.1 402 Payment Required
{
  "error": "quota_exceeded",
  "limit": 10, "used": 11, "metric": "daily_calls",
  "plan": "free",
  "upgradeUrl": "https://webhook.sp33c.tech/billing"
}
```

- Ingest already rejects over-limit and auto-disables on sustained 5× overage —
  we keep that, just make the response shape explicit and add the `upgradeUrl`.
- 402 (Payment Required) for plan-cap breaches; 429 for the burst guard.
- The webhook owner gets a notification ("You hit your daily limit on `<plan>`")
  — reuse the existing alert channels on the user row.

## Reflecting limits to the user (transparency)

A `GET /me/usage` endpoint returns `{ plan, limits, used }` for today so the
dashboard can render usage-vs-limit bars. See [../ui/usage-and-billing.md](../ui/usage-and-billing.md).
