# UI — usage & billing (reflecting the cost plan in the system)

This is the "reflect the cost plan in the system" requirement: the user can see,
everywhere it matters, which plan they're on and how close they are to its limits.

## Dashboard usage panel (web + app)

Source: `GET /me/usage` → `{ plan, limits, used }`.

Render per metric a **usage-vs-limit bar**:

```
Daily calls    ▓▓▓▓▓▓▓░░░  72 / 100      (Pro)
Daily data     ▓▓░░░░░░░░  3.1 / 20 MB
Webhooks       ▓▓▓░░░░░░░  3 / 25
Devices        ▓░░░░░░░░░  1 / 10
```

- Color shifts amber ≥80%, red at 100%.
- At/over a daily limit, show the **block reason** inline ("Daily call limit
  reached on Free — upgrade to keep receiving") with a deep link to `/billing`.
  This mirrors the 402 `quota_exceeded` payload from [../plans/limits.md](../plans/limits.md).
- Daily counters reset at UTC midnight (matches `dayKey()` in `ratelimit.ts`) —
  state that explicitly so the reset isn't a mystery.

## Plan strip / settings

- A persistent "Plan: Pro · renews 14 Jul" strip (from `/me/plan`).
- `planStatus`:
  - `active` → normal.
  - `past_due` → amber banner "Payment issue — update billing" → portal.
  - `canceled` → "Active until <renewsAt>, then Free."
- Manage button → `POST /billing/portal`.

## Where the plan is reflected

| Surface | Shows |
|---|---|
| Dashboard usage panel | live usage vs limits |
| Settings / plan strip | plan, status, renewal date, manage link |
| Webhook create | if at `maxWebhooks`, disable "New webhook" + upgrade hint |
| Device register | if at `maxDevices`, message + upgrade hint |
| Ingest failures | over-limit notification via existing alert channels |
| Plan picker | current plan badged |

## Consistency

All of the above read from the **same** `plans.ts`-derived limits the limiter
enforces — display and enforcement can never diverge. Localize copy via `i18n.ts`.
