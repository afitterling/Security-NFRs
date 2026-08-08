# Sandbox Testing, Transfer Behavior, and Gotchas

## Sandbox has parity (NFR-8)

RevenueCat fires webhooks for **sandbox** purchases too, with
`environment: "SANDBOX"`. The backend must **not** filter by environment — treat
`SANDBOX` and `PRODUCTION` identically — or the flow is untestable before launch.
"It's only a sandbox purchase" is **not** why the web is missing it; attribution
(previous doc) is.

## End-to-end test procedures

Sign into a **Sandbox tester** Apple ID on the device first (Settings → App Store →
Sandbox Account). Then:

**Path A — exercises `TRANSFER` (buy → then sign in):**
1. Sign **out** of the app account (next purchase is anonymous).
2. Sandbox-purchase → app unlocks Pro locally.
3. Sign **into** the app account → `Purchases.logIn(email)` re-attributes → RC emits
   `TRANSFER` → backend grants → **web shows Pro.**

**Path B — the happy path real users should hit (sign in → then buy):**
1. Sign into the account **first**.
2. Purchase → `INITIAL_PURCHASE` / `NON_RENEWING_PURCHASE` **with the email** →
   backend grants → web shows Pro. (No `TRANSFER` needed.)

Verify at each layer: app Settings ("Pro is active") → RC `active_entitlements` for
the email → backend flag → web signed-in.

## Transfer-behavior setting gates Path A

If Path A doesn't fire a `TRANSFER`, check **RevenueCat → Project settings →
transfer behavior**. It must be **"Transfer to the new App User ID"** (the default).
If set to "Keep with original," the anonymous purchase never moves and no event is
sent — Path B is then the only way.

## Local `.storekit` config files — a foot-gun for integration testing

A StoreKit configuration file (`Foo.storekit`) is great for pure-UI iteration but
**lies** about store integration:

- It only applies when the app is **launched from Xcode with the debugger
  attached** (⌘R). `expo run:ios`, a Release build, TestFlight, or tapping the icon
  all **ignore** it and hit the real store.
- The file must be **added to the Xcode project** (a `project.pbxproj` member) or
  the scheme's StoreKit-Configuration dropdown is empty / flaky. Expo `prebuild`
  regenerates `ios/` and **strips** the reference each time.
- Its product ids must match RC/ASC (`nilo_pro_*`, not `pro_*`) or the offering is
  empty even locally.
- **For real integration/webhook testing, use sandbox with `StoreKit
  Configuration = None`.** The local file can't produce webhooks or exercise
  attribution.

## Condensed gotchas

- **App shows Pro, web doesn't** → anonymous purchase (attribution), not sandbox.
  Check `…/customers/<email>/active_entitlements`.
- **`TRANSFER` ignored** → buy-then-sign-in silently lost. Handle it, and make the
  grant satisfy your "is-active" predicate (`lifetime:true` or a real expiry).
- **`CONFIGURATION_ERROR` / empty offerings** → config, not code. Paid-apps
  agreement → product-id match → **correct RC app** → credentials → propagation.
- **Two credential slots** — `app_store_connect_api_key_configured` **and**
  `subscription_key_configured`. Uploading one ≠ both.
- **Legacy key 403s on `/v2`** → mint a v2 `sk_…`.
- **Store ids are permanent** → fix RC/`.storekit` to match ASC, never the reverse.
- **Prices are in ASC**, not RevenueCat.
- **`expand=product`** is singular on packages.
- **A grant must read as active** — verify against the backend's entitlement
  predicate, not just "we called setPro".
- **Normalise the account id identically** on `logIn` and in the webhook mapper
  (Nilo lower-cases the email) or the map misses on case.
- **Rotate `sk_…` keys** shared in chats/logs.
