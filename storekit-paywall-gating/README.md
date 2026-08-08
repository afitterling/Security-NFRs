# StoreKit Paywall Gating (macOS, client-side)

Reusable spec for gating a premium feature behind a **StoreKit 2 non-consumable** in a native
macOS app, and testing it. Complements:
- `../app-store-iap-setup/` — App Store Connect product/price/API setup (server side).
- `../general-specs-payment/` — web payment flows (different surface).

This spec is the **client/app side**: how the app decides "unlocked", how the paid feature is
enforced, what a purchase returns, and the repeatable sandbox test procedure.

Reference implementation: PingTray (`swift/Sources/PingTray/Store.swift`,
`HostsWindowController.swift`, `main.swift`). Product `tech.sp33c.pingmonitor.customhosts`,
$2.99 one-time, unlocks user-defined monitored hosts; the default host stays free.

## Non-functional requirements
1. **Single source of truth for entitlement.** `isUnlocked` is derived *only* from
   `Transaction.currentEntitlements`. No cached UserDefaults "unlocked" flag — a flag is a
   free-unlock backdoor and can desync from the real entitlement.
2. **No debug/override backdoor in shippable builds.** Never gate unlock on a UserDefaults key
   or env var that survives in a release binary. (We tried `#if DEBUG` + `localUnlock`; even
   compiled-out it's a smell. Removed entirely — test with a real sandbox user instead.)
3. **Defense in depth on the paid action.** Disabling the UI is not enough. Every mutation
   path to the premium data must `guard isUnlocked`. See `gating-enforcement.md`.
4. **Free tier always works** with zero purchase and zero account.
5. **Restore is always reachable** (Apple guideline) — in our app from both the feature window
   and Settings; it calls `AppStore.sync()` then re-reads entitlements.

## Files
- `gating-enforcement.md` — the three-layer enforcement pattern + the single-write-path rule.
- `purchase-and-entitlement-data.md` — what StoreKit returns on purchase; no PII; appAccountToken.
- `sandbox-testing.md` — repeatable end-to-end test against StoreKit sandbox.
- `gotchas.md` — the traps we actually hit (sandbox purchase survives local wipe, etc.).
