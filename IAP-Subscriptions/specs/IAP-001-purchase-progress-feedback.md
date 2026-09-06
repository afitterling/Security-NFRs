# IAP-001 — Purchase progress & outcome feedback

- **Status:** Proposed
- **Group:** IAP & Subscriptions
- **Applies to:** Every app that sells an in-app purchase or subscription
  (StoreKit / Google Play Billing, directly or via RevenueCat).
- **Last updated:** 2026-08-09

## Feature

A purchase **MUST** be legible while it is happening. From the moment the user
taps *Buy* until the entitlement is actually active, the app **MUST** show what
state the purchase is in, and **MUST** resolve into exactly one clearly-stated
outcome — bought, cancelled, pending, or failed. The user must never be left
looking at a frozen button or a spinner that never ends, and must never be
tempted to tap *Buy* a second time.

## Why this is not covered by "the store shows a sheet"

The store's own payment sheet covers only the middle of the flow. Two windows
are the app's responsibility and are where the flow visibly dies:

```
 tap Buy ──► [app] ──► StoreKit/Play sheet ──► [app] ──────────► entitlement active
             ▲                                 ▲
             │ gap 1: sheet can take            │ gap 2: sheet is gone, but receipt
             │ seconds to appear — button       │ validation + RevenueCat webhook +
             │ looks dead, user taps again      │ backend flag still pending — app
                                                  looks like nothing happened
```

Gap 2 is the expensive one: the user has been charged, the sheet has dismissed,
and the app still shows the paywall. That reads as "I paid and got nothing".

## Requirement

1. **Immediate busy state.** The purchase control **MUST** enter a visible busy
   state on the same interaction that starts the purchase — disabled, plus a
   label/spinner change ("Purchasing…"). Every other purchase-initiating control
   on screen (other plan tiers, *Restore*) **MUST** be disabled for the duration.
2. **No double-charge path.** While a purchase is in flight the app **MUST NOT**
   dispatch a second purchase call, including via rapid re-taps, a re-rendered
   sheet, or a back-then-forward navigation.
3. **The post-sheet gap MUST be shown.** After the store sheet dismisses and
   while the app is still validating/awaiting the entitlement, the app **MUST**
   display a distinct in-progress state ("Completing purchase…"), not the
   pre-purchase paywall and not a bare success claim.
4. **Success is declared off the entitlement, not off the call returning.** The
   success state **MUST** be driven by the entitlement actually flipping
   (`Transaction.currentEntitlements` / RevenueCat `CustomerInfo` listener), the
   same single source of truth all gating uses. A purchase call that returns
   without an active entitlement is **not** a success.
5. **Cancellation is not an error.** A user-cancelled purchase (`userCancelled`
   / Play `BILLING_RESPONSE_RESULT_USER_CANCELED`) **MUST** return the UI
   silently to its idle state, or say so neutrally. It **MUST NOT** raise an
   error dialog, error copy, or an error-level log/alert.
6. **Pending/deferred is its own outcome.** *Ask to Buy* (Family Sharing),
   SCA challenges and other deferred approvals **MUST** be reported as pending —
   "waiting for approval", entitlement not yet granted — and **MUST NOT** be
   reported as either success or failure. The app **MUST** grant access when the
   approval later arrives, without requiring a repeat purchase.
7. **Failure is terse and actionable**, and **MUST** distinguish at minimum:
   store/network failure (retryable), and already-owned (→ route to *Restore*,
   see [restore](../../general-specs-payment/ui/restore-purchases.md)) — never a
   raw store error code as the user-facing message.
8. **Bounded wait.** The in-progress state **MUST** have an escape: a timeout
   that resolves to a stated outcome, and a way to dismiss that never silently
   discards a paid-for entitlement (dismissing returns to a screen that still
   reflects the real entitlement state once it lands).
9. **Status survives backgrounding.** The store sheet, Apple-ID authentication
   and *Ask to Buy* all background the app. In-progress state **MUST** be
   restored on foreground and resolved — a purchase completed while backgrounded
   **MUST** still surface its outcome.
10. **Copy is specific.** Status and outcome strings **MUST** name the product or
    entitlement ("Nilo Pro is active"), not a generic "Success", and **MUST** be
    localized ([I18N-001](../../general-specs/internationalization/I18N-001-localization.md)).

## Rationale

Purchase is the one flow where invisible latency costs money and trust at the
same time. The charge is irreversible from the user's point of view, the
validation chain (StoreKit → RevenueCat → webhook → backend flag) is genuinely
multi-second, and the failure modes are indistinguishable to the user without
being told apart: cancelled, deferred, failed and "already yours" all look like
"nothing happened". Silence there produces double purchases, refund requests and
support tickets that no amount of paywall polish offsets.

Declaring success off the entitlement rather than off the call returning is the
same rule the rest of this bundle enforces for gating — see
[`../storekit-paywall-gating/`](../storekit-paywall-gating/README.md) NFR-1 and
[`../revenuecat-integration/`](../revenuecat-integration/README.md) NFR-1 — applied to
the UI layer, so the progress indicator and the gate can never disagree.

## Acceptance criteria

- [ ] Tapping *Buy* disables that control and every sibling purchase/restore control immediately.
- [ ] Rapid double-tap on *Buy* produces exactly one store transaction.
- [ ] After the store sheet dismisses, a "Completing purchase…" state is visible until the entitlement resolves.
- [ ] The success state appears only after the entitlement is active, and names the product.
- [ ] Cancelling the store sheet returns to idle with no error dialog and no error-level log.
- [ ] An *Ask to Buy* purchase shows a pending state, and access is granted later without re-purchasing.
- [ ] A forced network failure mid-purchase yields a terse retryable message, not a store error code.
- [ ] Purchasing an already-owned product points the user at *Restore* rather than failing opaquely.
- [ ] Backgrounding during the store sheet and returning still resolves to a stated outcome.
- [ ] No path leaves a spinner running with no timeout and no dismissal.

## Implementation notes

- **Shape it as one state machine, not scattered booleans:**
  `idle → purchasing → verifying → (active | cancelled | pending | failed)`.
  A single `purchaseState` in the store (`store.tsx`) drives both the button and
  the status surface, so they cannot disagree.
- **Drive `active` from the entitlement listener**, not from the `purchase()`
  promise — the listener is already wired for gating
  (`addCustomerInfoUpdateListener` / `Transaction.updates`), so the same event
  that unlocks the feature ends the progress state.
- **Detect cancellation explicitly** — RevenueCat surfaces
  `userCancelled` on the error; StoreKit 2 returns `.userCancelled` as a *result*,
  not a thrown error. Branch on it before any generic catch, or cancellation
  falls into the failure path (the mirror of the "`false` ≠ error" rule in
  [restore-purchases](../../general-specs-payment/ui/restore-purchases.md)).
- **`.pending`** (StoreKit 2 deferred / Ask to Buy) is a distinct result case —
  handle it alongside `.success` and `.userCancelled`, never in `default`.
- **Dev builds can't exercise this.** With the store key unset, purchase calls
  short-circuit; verify against a TestFlight/prod build with a sandbox account —
  see [`../revenuecat-integration/sandbox-and-gotchas.md`](../revenuecat-integration/sandbox-and-gotchas.md)
  and [`../storekit-paywall-gating/sandbox-testing.md`](../storekit-paywall-gating/sandbox-testing.md).
- Sandbox makes gap 2 *longer*, not shorter — it is the honest place to test the
  "Completing purchase…" state.
