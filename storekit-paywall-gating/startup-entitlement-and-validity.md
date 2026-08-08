# Startup entitlement check & transaction validity

Answers: at app start, do I just re-read `Transaction.currentEntitlements`, and how do I know
a transaction is *valid*? Yes — re-read entitlements; StoreKit 2 tells you validity via the
`VerificationResult`. For a no-backend app this is all you need.

## When to check
Two triggers, both wired in `Store.start()`:
1. **At launch** — call `refreshEntitlement()` once. The app must reflect the real entitlement
   on every cold start (don't trust a cached "unlocked" bool — there isn't one).
2. **Continuously** — a `Transaction.updates` listener for the app's lifetime, so a purchase,
   refund, or revocation that lands *while running* re-evaluates immediately:
   ```swift
   updatesTask = Task.detached {
       for await update in Transaction.updates {
           if case .verified(let t) = update { await t.finish(); await self.refreshEntitlement() }
       }
   }
   ```
Both funnel through one recompute; `onChange` then refreshes UI and feature gates.

## How you know a transaction is valid
Each element of `Transaction.currentEntitlements` / `Transaction.updates` is a
**`VerificationResult<Transaction>`**. StoreKit 2 has *already* checked the JWS signature
against Apple's certificate chain **on-device** — you do not verify crypto yourself:

- `.verified(transaction)` → authentic. Trust it.
- `.unverified(transaction, error)` → signature/authenticity check **failed**
  (`VerificationResult.VerificationError`). Do **not** grant. Never read the payload out of an
  `.unverified` case as if it were trustworthy.

So "is this transaction valid?" = "did it arrive as `.verified`?". That's the whole answer for
on-device gating.

## Business checks (after `.verified`)
Authenticity ≠ entitlement. Once verified, confirm it actually grants *your* feature:
- `productID == Self.productID` — it's the right product.
- `revocationDate == nil` — not refunded/charged back (a refund re-locks the app).
- `ownershipType` — only if family-sharing changes behavior.
- (Non-consumable: no expiry. **Subscriptions** would also check `expirationDate` and
  `isUpgraded`.)

## The exact pattern (Store.swift)
```swift
func refreshEntitlement() async {
    var unlocked = false
    for await result in Transaction.currentEntitlements {
        if case .verified(let t) = result,
           t.productID == Self.productID,
           t.revocationDate == nil {
            unlocked = true
        }
    }
    await MainActor.run { self.isUnlocked = unlocked; self.onChange?() }
}
```

## Offline behavior
`currentEntitlements` reads on-device synced state and verifies locally — it **works offline**,
no network round-trip. Call `AppStore.sync()` **only** on an explicit user "Restore" (it
re-pulls transactions from the Apple Account and can prompt for sign-in). Do not sync on every
launch.

## PingTray specifics — no backend
PingTray has **no server**, so:
- **Do not persist transaction IDs.** `currentEntitlements` at launch is the source of truth.
- On-device `.verified` is sufficient proof; no server validation needed.
- Restore is available but not required for normal unlock — entitlements already sync per
  Apple Account.

## When to escalate to server validation
Only if a backend must independently confirm entitlement (cross-platform accounts, server-side
feature flags, fraud checks). Then use the **App Store Server API**
`GET /inApps/v1/transactions/{transactionId}` and verify the returned JWS — see
[`purchase-and-entitlement-data.md`](./purchase-and-entitlement-data.md). Not needed here.

## Anti-patterns
- ❌ Caching an `unlocked` Bool in UserDefaults (free-unlock backdoor; can desync). See
  [`gating-enforcement.md`](./gating-enforcement.md).
- ❌ Trusting `.unverified`.
- ❌ Treating a bare transaction-ID string as proof — proof is the *verified signed payload*,
  not the id.
- ❌ Calling `AppStore.sync()` on every launch.
