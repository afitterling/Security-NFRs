# What you get back from an App Store purchase / "login"

Short answer: **no personal identity.** Apple never hands the app the buyer's name, email, or
Apple ID. You get a signed transaction describing *what* was bought, not *who* bought it.

## There is no "login" surface
The App Store does not provide user login to your app. A purchase is not a sign-in. If you
actually need user identity, that is **Sign in with Apple** (a separate feature), which returns
a stable opaque user identifier and — only on the first consent — name and an (optionally
private-relay) email. We do not use it; the app is account-free.

## StoreKit 2 purchase return shape
`try await product.purchase()` → `Product.PurchaseResult`:
- `.success(VerificationResult<Transaction>)` — verify, then read the `Transaction`.
- `.userCancelled`
- `.pending` (e.g. Ask to Buy / SCA)

The verified `Transaction` carries:
| field | meaning |
|---|---|
| `productID` | which product (`tech.sp33c.pingmonitor.customhosts`) |
| `id` | this transaction's id |
| `originalID` | first transaction id (same as `id` for a fresh non-consumable) |
| `purchaseDate` / `originalPurchaseDate` | when |
| `ownershipType` | `.purchased` vs `.familyShared` |
| `revocationDate` | nil unless refunded/revoked — gate on `== nil` |
| `environment` | `.sandbox` / `.production` (Xcode/StoreKit-test = `.xcode`) |
| `appAccountToken` | the UUID *you* supplied at purchase, if any |
| `jsonRepresentation` / JWS | signed payload for server-side validation |

## Tying a purchase to your own user
Apple gives you no identity, so if you must correlate to your backend user, generate a UUID and
pass it as `appAccountToken` in the purchase options; it comes back on the `Transaction` and in
the App Store Server Notifications. Otherwise the entitlement is simply device/Apple-ID bound
and `Transaction.currentEntitlements` is all you need (our case — no backend).

## Server-side (if/when needed)
- **App Store Server API** — query/look up transactions and entitlements by transaction id.
- **App Store Server Notifications v2** — Apple POSTs signed JWS events (PURCHASE, REFUND,
  REVOKE…) to your endpoint. Still no PII; keyed by transaction/original-transaction id.

## Implication for privacy disclosures
Because no PII is returned, the app can honestly answer App Store privacy as "Data Not
Collected" for the purchase flow. Keep this consistent with the privacy policy.
