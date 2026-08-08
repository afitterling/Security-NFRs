# RevenueCat Integration — Entitlement Sync NFR

**When to apply:** any project that uses **RevenueCat** to sell an entitlement on a
device *and* mirrors that entitlement to a backend (for cross-device sync, a web
"is this user premium?" check, or server-gated features). If RevenueCat is in the
stack, these are the non-functional requirements and the traps to check first.

This is the **runtime/integration** side. It complements, and does not repeat:
- `../app-store-iap-setup/` — one-time API-driven *setup* (create products, prices,
  wire the RC app/products/entitlement/offering). Read that to stand things up.
- `../storekit-paywall-gating/` — client-side gating with **direct StoreKit 2** (no
  RevenueCat). Different mechanism; same *don't-trust-a-cached-flag* philosophy.

> Status: **proven, and hard-won.** Every requirement and gotcha below comes from a
> real Nilo incident where the paywall showed empty and/or purchases never reached
> the web. Worked example throughout: **Nilo** (bundle `tech.sp33c.nilo`, RC project
> `sp33c`, App Store RC app `app0588a3a726`, entitlement `pro`, backend
> `sst/src/revenuecat.ts` + `iosapp/src/purchases.ts`).

## The mental model (memorise this)

```
 device purchase ──► RevenueCat (system of record) ──► webhook ──► backend `pro` flag
     │                      ▲                                            │
     │  Purchases.logIn(email) sets app_user_id = account id            ▼
     └─────────► local entitlement (app shows Pro)          web / sync read the flag
```

Two independent planes: the **device** shows Pro from its *local* RevenueCat
entitlement; the **backend/web** only ever knows Pro from a **webhook that maps to
an account**. They desync the moment a purchase isn't attributed to an account.
Most "the app shows Pro but the web doesn't" bugs live in that gap.

## Non-functional requirements (locked)

1. **The webhook is the only source of backend truth.** The server sets its `pro`
   flag *exclusively* from verified RevenueCat webhooks. Never let the device
   POST "I'm Pro" — that's a free-unlock backdoor. (Mirrors the StoreKit spec's
   NFR-1: entitlement derives from the store, never a cached flag.)
2. **Every purchase must be attributed to an account.** On sign-in the app must
   call `Purchases.logIn(<account-id>)` so `app_user_id` = the account identifier
   (Nilo uses the lower-cased email). An **anonymous** purchase never reaches the
   backend — the webhook has no account to map to. See
   [attribution-and-webhook-sync.md](attribution-and-webhook-sync.md).
3. **Handle the whole event set, including `TRANSFER`.** Buy-then-sign-in aliases
   an anonymous purchase to the account and fires a **`TRANSFER`** event, *not* a
   fresh purchase. Ignoring it silently drops the entitlement. This is the single
   most common attribution bug.
4. **Webhook is authenticated and always acks `2xx`.** Constant-time compare a
   shared secret; once authorized, return `2xx` even on data you choose to ignore,
   so RevenueCat doesn't retry-storm on your own quirks.
5. **RC `store_identifier` == the store product id, on the *correct* RC app.** A
   project holds many apps (App Store / Play / Web Billing / Test). Products, the
   entitlement, and the offering packages must be wired on the app the running
   SDK key maps to. Any mismatch → empty offerings. See
   [offerings-and-credentials.md](offerings-and-credentials.md).
6. **"Offerings are empty" is a *configuration* error, not a code bug.** When
   `getOfferings()` returns nothing, RevenueCat reports `CONFIGURATION_ERROR`;
   walk the checklist (Paid Apps Agreement, product-id/app mismatch, credentials,
   propagation, local StoreKit config) before touching code.
7. **Both Apple credential slots must be configured.** The **App Store Connect API
   key** *and* the **In-App Purchase key** — both dashboard-only. Without them
   receipts don't validate / product metadata won't sync.
8. **Sandbox has parity — do not filter by environment.** Webhooks fire for
   sandbox purchases too; the backend must treat `SANDBOX` and `PRODUCTION`
   identically, or you can't test the flow. See
   [sandbox-and-gotchas.md](sandbox-and-gotchas.md).

## Files

| Area | Doc |
|---|---|
| Attribution (`app_user_id`), the webhook→`pro` flow, event mapping incl. `TRANSFER` | [attribution-and-webhook-sync.md](attribution-and-webhook-sync.md) |
| Why offerings come back empty (`CONFIGURATION_ERROR` checklist), credentials, v1/v2 keys, multi-app | [offerings-and-credentials.md](offerings-and-credentials.md) |
| Sandbox test procedure, transfer-behavior setting, local `.storekit` traps, condensed gotchas | [sandbox-and-gotchas.md](sandbox-and-gotchas.md) |
