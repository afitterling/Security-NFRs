# IAP-003 — RevenueCat integration baseline

- **Status:** Proposed
- **Group:** IAP & Subscriptions
- **Applies to:** Every app that has RevenueCat in its stack. That covers a device
  SDK (`react-native-purchases`, `RevenueCat` Swift, `purchases-js`), and covers a
  backend that mirrors the entitlement, whether or not it also uses StoreKit or Play
  Billing directly.
- **Last updated:** 2026-10-03

## Feature

RevenueCat is the layer that **attributes** a store purchase to an account and
**relays** the resulting entitlement to our backend. It is not where money or prices
live (that is the store), and it is not something the client can talk past. Every
app that uses it **MUST** configure it the same way: keys in the right places,
purchases attributed to a stable account id, exactly one code path finishing
transactions, and a backend that learns about entitlements only from verified
RevenueCat data.

This spec turns the locked NFRs of the
[`revenuecat-integration/`](../revenuecat-integration/README.md) playbook into
numbered requirements with acceptance criteria. It also adds the rules that the
playbook does not yet cover: identity lifecycle, transaction finishing, ordering,
reconcile, deletion and observability. The playbook stays the how-to.

```
 store (money, prices) ──► RevenueCat (attribution, relay) ──► webhook ──► backend entitlement
        ▲                          ▲                                            │
        │ one finisher (§9)        │ app_user_id = account id (§5–§8)           ▼
   device purchase ──────────► CustomerInfo listener ──► gates (IAP-004)   web / server gates
```

## Requirement

### Configuration

1. **Keys sit where their class belongs.** The public SDK key (`appl_…`, `goog_…`,
   `rcb_…`) **MUST** be injected per stage from build configuration
   ([DEL-003](../../general-specs/delivery/DEL-003-per-environment-app-identity.md)).
   The v2 secret key (`sk_…`) and the webhook authorization secret **MUST** exist
   only server-side as SST Secrets
   ([SEC-001](../../general-specs/security/SEC-001-secrets-management.md)). They
   **MUST NOT** appear in a client bundle, the repo or a log line. A secret pasted
   into a chat or a log **MUST** be rotated.
2. **Wired on the right RC app, and verified before release.** Products, the
   entitlement, and the current offering's packages **MUST** sit on the RC app that
   the shipped SDK key maps to. A **pre-release check** (script against the v2 API)
   **MUST** assert the following. The current offering is non-empty. Every package's
   `product.app_id` is the store app for that platform. Every product's
   `store_identifier` exists in the store. Both Apple credential slots are configured
   (`app_store_connect_api_key_configured`, `subscription_key_configured`).
3. **Gate on entitlements, never on products.** Client and server code **MUST**
   decide access from **entitlement identifiers** (`pro`). They **MUST NOT** decide
   it from product or package ids. Product ids change with terms, prices and
   promotions, while the entitlement is the stable contract. The mapping from
   entitlement to capabilities lives in one place
   ([IAP-004](IAP-004-payment-gating-freemium-throttling-sync.md) §1).
4. **Identifiers are permanent.** Entitlement, product and offering identifiers
   **MUST** be chosen once and never renamed. A rename is a new identifier plus a
   migration.
5. **The paywall is built from the offering.** The packages shown **MUST** come
   from `getOfferings().current`, and prices **MUST** come from the SDK's localized
   price strings ([IAP-002](IAP-002-price-display-fidelity.md) §7). An empty or
   failed offering **MUST** render a stated "plans unavailable, try again" state,
   never a blank sheet or a crash. It **MUST** also be logged as a configuration
   error, not a code error.

### Identity

6. **`app_user_id` is the stable account id.** In apps with accounts, the
   `app_user_id` **SHOULD** be the opaque account identifier (Cognito `sub` or our
   own user id), not the email address. An email is personal data sent to a third
   party, and it changes when the user changes it. Apps that already use the email
   **MUST** normalize it the same way (trim, lower-case) in `logIn` and in the
   webhook mapper, and **SHOULD** migrate.
7. **The identity lifecycle is complete.** The app **MUST** call `logIn(<id>)` on
   sign-in and on session restore, before any purchase UI can show. It **MUST**
   call `logOut()` on sign-out, so the next account on the device does not inherit
   the entitlement. Account deletion
   ([PRIV-002](../../general-specs/privacy/PRIV-002-gdpr-dsgvo-user-rights.md))
   **MUST** also delete the RevenueCat customer through the REST API.
8. **The transfer behaviour is set and handled.** The RC project setting **MUST**
   be "Transfer to the new App User ID", and the backend **MUST** handle `TRANSFER`
   (§16). Buying before signing in is the most common way an entitlement gets lost.
9. **Account-free apps stay anonymous.** An app without accounts **MAY** run on
   RC's anonymous id. It then has no backend mirror, gates purely on device
   (`CustomerInfo` or StoreKit), and **MUST** offer Restore as the way to move a
   purchase to a new device.

### On the device

10. **Exactly one transaction finisher.** Only one code path in the app **MAY**
    finish store transactions. If the app also uses StoreKit or Play Billing
    directly, for example for consumables or other stores listening on
    `Transaction.updates`, RevenueCat **MUST** be configured with
    `purchasesAreCompletedBy: .myApp`. The app then reports each verified purchase
    with `recordPurchase` / `syncPurchases`. Two finishers on one stream let a
    consumable be finished before its grant, and the user pays and gets nothing.
11. **One entitlement listener drives UI and gates.** The `CustomerInfo` stream
    (`customerInfoStream` / `addCustomerInfoUpdateListener`) **MUST** feed the same
    state that every gate and the purchase-progress UI read
    ([IAP-001](IAP-001-purchase-progress-feedback.md) §4). The SDK's own cache
    **MAY** serve offline launches. The app **MUST NOT** persist its own
    `isPro`/`unlocked` flag.
12. **Restore is always reachable.** It calls `restorePurchases()`, and "nothing to
    restore" is a neutral message, not an error.
13. **A build without a key never grants.** With the SDK key unset (dev, CI,
    screenshot builds), purchase paths **MUST** short-circuit to a visible
    "purchases unavailable in this build". They **MUST NOT** grant access silently,
    and **MUST NOT** fall back to a hidden override in a release configuration.

### Backend

14. **Verified RevenueCat data is the only source of backend truth.** The server
    sets its entitlement **exclusively** from authenticated webhooks and its own
    REST reads (§19). No endpoint **MAY** accept "I'm Pro" from a device.
15. **The webhook is authenticated, and acks once authorized.** The `Authorization`
    header **MUST** be compared in constant time with a length guard, and a failure
    **MUST** return `401`. Once a request is authorized, the handler **MUST** return
    `2xx` for every event: ignored types, unknown ids and test pings included. That
    way RevenueCat retries only real failures.
16. **The whole event set is mapped, and a grant reads as active.**
    `INITIAL_PURCHASE`, `RENEWAL`, `PRODUCT_CHANGE`, `UNCANCELLATION`,
    `NON_RENEWING_PURCHASE` and `TRANSFER` (to `transferred_to`) grant access.
    `EXPIRATION`, `REFUND` and `SUBSCRIPTION_PAUSED` revoke it. `CANCELLATION` is
    **not** a revoke: access lasts until `EXPIRATION`. `BILLING_ISSUE` keeps access
    through the store's grace period. Every grant **MUST** satisfy the backend's
    is-active predicate (see the
    [`TRANSFER` trap](../revenuecat-integration/attribution-and-webhook-sync.md)).
17. **The webhook is idempotent and order-safe.** Each event **MUST** be processed
    at most once, keyed on `event.id`. An event older than the last one applied to
    that customer (`event_timestamp_ms`) **MUST NOT** overwrite newer state. The
    preferred shape is to treat the webhook as a *trigger*, re-read the customer
    from the REST API, and store that. This removes ordering bugs altogether.
18. **Sandbox has parity, and is labelled.** The handler **MUST NOT** filter out
    `SANDBOX`, because that would leave the flow untestable. It **MUST** store the
    environment on the record, so support and metrics can tell TestFlight grants
    from paid ones.
19. **A reconcile path exists.** An authenticated, rate-limited endpoint (e.g.
    `POST /billing/sync`) **MUST** re-read the caller's customer from RevenueCat
    and update the stored entitlement. It serves the post-purchase UI before the
    webhook lands, and lets support answer "I paid but…".
20. **The endpoint is hardened.** The webhook route is POST only, has a body-size
    cap and no CORS, and sits behind the app's edge rate limit
    ([SEC-006](../../general-specs/security/SEC-006-edge-rate-limiting.md)).

### Privacy and operations

21. **Minimum data to RevenueCat.** Subscriber attributes **MUST NOT** carry name,
    email, phone or free text unless a documented feature needs them. RevenueCat's
    ad-network and attribution integrations **MUST** stay disabled, so the "no
    tracking" claim holds ([PRIV-003](../../general-specs/privacy/PRIV-003-tracking-consent-att.md),
    [`non-tracking-purchases/`](../non-tracking-purchases/README.md)). RevenueCat
    **MUST** be named as a processor on `/privacy`
    ([PAGE-001](../../general-specs/pages/PAGE-001-privacy-route.md)).
22. **Drift is observable.** The backend **MUST** emit metrics and alarm
    ([REL-001](../../general-specs/reliability/REL-001-observability-and-alerting.md))
    on webhook `401` spikes, handler `5xx`, and events whose `app_user_id` maps to
    no account. The last one is the signature of anonymous purchases.

## Rationale

Every RevenueCat failure we have had was silent: an empty paywall, a purchase that
unlocked the phone but never reached the web, a consumable finished by the wrong
listener. In each case the code "worked", because the money moved and the store was
happy, while the entitlement fell between two planes that nobody compared. The
rules above close each of those gaps once, by construction:

- Attribution and `TRANSFER` close the device-to-account gap.
- A single finisher closes the race on the transaction stream.
- A webhook-only backend closes the free-unlock backdoor.
- Order safety and reconcile close the "webhook arrived late or twice" gap.
- The pre-release check moves the most common failure, configuration, out of the
  field and into CI.

## Acceptance criteria

- [ ] No `sk_…` key or webhook secret is present in any client bundle or in the repo (grep of the built artifact).
- [ ] The pre-release check passes: non-empty offering, packages on the store app, both Apple credential slots configured.
- [ ] No client or server gate references a product or package id. All of them reference entitlement ids or capabilities.
- [ ] An empty offering shows "plans unavailable" with retry and logs a configuration error.
- [ ] Sign in, then sign out, then sign in as another account on the same device: the second account does not see the first account's entitlement.
- [ ] Buy anonymously, then sign in: `TRANSFER` grants the account, and the web shows Pro.
- [ ] Deleting an account deletes the RevenueCat customer.
- [ ] In an app that also uses StoreKit directly, a consumable purchase is credited exactly once, and RevenueCat never finishes the transaction.
- [ ] A build without an SDK key cannot reach an unlocked state.
- [ ] A webhook with a wrong secret gets `401`. An authorized unknown event type gets `2xx`.
- [ ] Replaying the same event id changes nothing. Delivering `RENEWAL` before an older `EXPIRATION` leaves the user active.
- [ ] `CANCELLATION` leaves access intact until `EXPIRATION`.
- [ ] `POST /billing/sync` updates the stored entitlement without waiting for a webhook.
- [ ] Alarms exist for webhook 401s, 5xx and unmapped `app_user_id`s.
- [ ] No email or name is set as a subscriber attribute, and no ad-network integration is enabled in the RC project.

## Implementation notes

- **Nilo** is the reference for webhook mapping (`sst/src/revenuecat.ts`,
  `evaluateRcEvent`, including `TRANSFER`). **Gap:** `app_user_id` is the email
  (§6), and there is no order guard or event-id dedupe yet (§17).
- **Dont Forget Me** (`swift/Sources/Store/ProStore.swift`) is the reference for
  §10. RevenueCat is configured with `purchasesAreCompletedBy: .myApp`, because
  `UnlockStore` and `AITripsStore` already listen on `Transaction.updates`, and a
  third finisher could finish the `aitrips` consumable before the credit was granted.
  The paywall works with RevenueCat unconfigured (§13).
- **WebhookNotification** follows the web + app + webhook shape in
  [`general-specs-payment/checkout/revenuecat.md`](../../general-specs-payment/checkout/revenuecat.md),
  including `GET /billing/sync` (§19).
- **OpenCycle, OpenOutdoor, Priorize, TimeWarp, Timer** ship
  `react-native-purchases` and have not been audited against this spec.
- **RC API calls for §2:** `GET /v2/projects/{p}/apps`,
  `GET …/offerings/{current}/packages?expand=product` (`expand` is singular), and
  `GET …/apps/{id}` for the credential flags. The recipes are in
  [`app-store-iap-setup/revenuecat-api.md`](../app-store-iap-setup/revenuecat-api.md).
- **Customer deletion (§7):** `DELETE https://api.revenuecat.com/v1/subscribers/{app_user_id}`
  with the secret key. This runs as part of the account-deletion job, not from the
  client.
