# Attribution + Webhook → Backend Entitlement

How a purchase travels from the device to a server-side `pro` flag, and the two
places it silently falls off the rails: **attribution** and **`TRANSFER`**.

## 1. Attribution — `app_user_id` must be the account id

RevenueCat identifies a buyer by `app_user_id`. Before the user signs in it's an
**anonymous** id (`$RCAnonymousID:…`). The webhook maps a purchase to *your*
account by reading that id, so the device must set it to the account identifier on
sign-in.

Nilo (`iosapp/src/purchases.ts`):
```ts
export async function identifyPurchaser(email: string): Promise<void> {
  if (!purchasesAvailable()) return;
  await initPurchases();
  await P().logIn(email.trim().toLowerCase()); // app_user_id = account email
}
// called from the store on sign-in / session restore
```

**Failure mode (real):** user buys **before** signing into their account → the
purchase is filed under the anonymous id → the app shows Pro (local entitlement)
but the webhook has no email → the account `pro` flag is never set → **web/sync
never see it.** Diagnosis is fast: query RC for the account and check active
entitlements — if the email has none, the purchase is anonymous.

```
GET https://api.revenuecat.com/v2/projects/<PROJ>/customers/<url-enc-email>/active_entitlements
→ items: []   ⟹  purchase is anonymous / not attributed to this account
```

**Requirement:** normalise the account id **the same way on both ends** (Nilo
lower-cases + trims the email in `logIn` *and* in the webhook's `pickEmail`), or
the map misses on case alone.

## 2. The webhook handler

RevenueCat POSTs entitlement changes to your endpoint. Contract (Nilo
`sst/src/index.ts` `POST /revenuecat` + `sst/src/revenuecat.ts`):

- **Authenticate** with a shared secret in the `Authorization` header, **length-
  guarded constant-time compare** (`timingSafeEqual`). Set the same value in the
  RC dashboard (Integrations → Webhooks → Authorization header) and in the backend
  env/secret.
- **Always ack `2xx` once authorized** — test pings, unmapped ids, event types you
  ignore: all return `{ ok: true }`. A non-2xx makes RevenueCat **retry**, so only
  reject on auth failure.
- **Map event → account → grant/revoke**, then persist.

```ts
const { email, action } = evaluateRcEvent(event);
if (email) {
  if (action.kind === "grant")  await setProEntitlement(email, { lifetime, expiresAt });
  else if (action.kind === "revoke") await clearProEntitlement(email);
}
return c.json({ ok: true });
```

## 3. Event → action mapping (the whole set)

| RC event type | Action | Notes |
|---|---|---|
| `INITIAL_PURCHASE`, `RENEWAL`, `PRODUCT_CHANGE`, `UNCANCELLATION` | **grant** | trust `expiration_at_ms`; `period_type==="LIFETIME"` ⟹ lifetime |
| `NON_RENEWING_PURCHASE` | **grant (lifetime)** | the one-time non-consumable; never expires |
| **`TRANSFER`** | **grant** to `transferred_to` | see §4 — the easy one to miss |
| `EXPIRATION`, `REFUND`, `SUBSCRIPTION_PAUSED` | **revoke** | |
| `CANCELLATION` | **ignore** | auto-renew off ≠ lost access; user keeps Pro until `EXPIRATION` |
| `BILLING_ISSUE`, `TEST`, others | **ignore** | no reliable grant/revoke; leave stored value |

## 4. `TRANSFER` — the buy-then-sign-in case (NFR-3)

When `logIn` attaches an anonymous user's purchases to an identified account,
RevenueCat **re-attributes** them and emits a **`TRANSFER`** event — not a fresh
`INITIAL_PURCHASE`. Its shape is different: no single `app_user_id`, no product,
no expiry — just the two sides:

```json
{ "event": { "type": "TRANSFER",
             "transferred_from": ["$RCAnonymousID:abc…"],
             "transferred_to":   ["user@example.com"] } }
```

Miss this and every buy-then-sign-in purchase vanishes server-side. Handle it
(Nilo `evaluateRcEvent`):

```ts
case "TRANSFER":
  return {
    email: firstEmail(event.transferred_to ?? []), // recipient = the account
    action: { kind: "grant", lifetime: true, expiresAt: null },
  };
```

**Why grant non-expiring here:** the event carries *no* product or expiry, so you
can't tell a subscription from a lifetime. Grant in a form your data model reads as
**active**, and let the follow-up events RevenueCat sends **to the new id**
reconcile the real term:
- a still-active sub keeps Pro until its `EXPIRATION` revokes it,
- a `RENEWAL` overwrites the stored expiry with the real one,
- a genuine lifetime just stays.

⚠️ **Data-model trap:** a "grant" is only Pro if it reads as active. Nilo's
`entitlementOf` is Pro **iff** `proLifetime===true` **or** `proExpiresAt > now`. A
grant with `lifetime:false, expiresAt:null` sets *neither* ⟹ **not Pro**. That's
why `TRANSFER` grants `lifetime:true` rather than an empty grant. Whatever your
model, make the `TRANSFER` grant satisfy the "is-active" predicate.

`transferred_from` is normally the anonymous id (owns no account) — nothing to
revoke there. If you support account→account moves, revoke from any *email* in
`transferred_from`.

## 5. Verify end-to-end

- **RC:** `…/customers/<email>/active_entitlements` shows the entitlement.
- **Backend:** the account row has the flag set (Nilo: DynamoDB PROFILE `pro` /
  `proLifetime` / `proExpiresAt`; or hit the app's `/entitlement` endpoint).
- **Web:** signed in with the same account → Pro renders.
