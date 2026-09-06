# IAP & Subscriptions

**In-app purchases only** — the store-mediated purchase of a subscription or
non-consumable, and everything that hangs off it: setting the products up,
attributing the purchase, gating on the entitlement, telling the user what the
purchase is doing, and keeping the privacy label honest.

Web/card checkout (Stripe, PayPal, Polar, merchant-of-record, plan catalogs,
billing API) is **not** here — that's
[`../general-specs-payment/`](../general-specs-payment/README.md).

## Specs

Numbered NFRs, same conventions as [`../general-specs/`](../general-specs/) —
stable `IAP-NNN` IDs, RFC 2119 keywords, acceptance criteria.

| ID | Spec | Status |
|---|---|---|
| [IAP-001](specs/IAP-001-purchase-progress-feedback.md) | Purchase progress & outcome feedback — show what the purchase is doing while it's in flight; resolve into exactly one stated outcome (bought / cancelled / pending / failed). | _Proposed_ |
| [IAP-002](specs/IAP-002-price-display-fidelity.md) | Price display fidelity — every price shown outside the store UI is generated from the store's own price schedules, never hand-maintained. | _Proposed_ |

## Playbooks

Ordered the way you'd hit them building an app:

| Folder | What it covers |
|---|---|
| [`app-store-iap-setup/`](app-store-iap-setup/README.md) | One-time, **API-driven setup**: App Store Connect subscription groups, prices across all ~175 territories, free trials, RevenueCat products/entitlement/offering, per-country prices on the landing page. |
| [`revenuecat-integration/`](revenuecat-integration/README.md) | **Runtime entitlement sync**: `app_user_id` attribution, webhook → backend `pro` flag (incl. `TRANSFER`), why offerings come back empty, the multi-app trap. |
| [`storekit-paywall-gating/`](storekit-paywall-gating/README.md) | **Client-side gating** with direct StoreKit 2 (macOS): entitlement as single source of truth, defense-in-depth on the paid action, restore, sandbox testing. |
| [`non-tracking-purchases/`](non-tracking-purchases/README.md) | **ATT vs. App Store privacy label** (Guideline 5.1.2(i)) — collecting purchase data is not tracking; the label and the shipped binary must agree. |

## How they relate

```
 app-store-iap-setup ──► products, prices, trials exist in the stores
          │
          ▼
   user taps Buy ──► IAP-001 (progress & outcome feedback)
          │
          ▼
 revenuecat-integration ──► purchase attributed to an account,
          │                  webhook sets the backend entitlement
          ▼
 storekit-paywall-gating ──► the client gates the feature off that entitlement
          │
          ▼
 non-tracking-purchases ──► the privacy label & shipped binary must match all of it
```

Setup happens **once per app**; the purchase flow, gating and label get
re-verified **every release**.

## Shared rules across all of these

- **Entitlement is derived from the store, never from a cached flag** — no
  UserDefaults "unlocked", no device POSTing "I'm Pro". A flag is a free-unlock
  backdoor. This is also what "success" means in IAP-001.
- **Cancellation is not an error.** Neither is "no purchase found" on restore.
- **Sandbox has parity with production.** Don't filter webhooks by environment,
  or the flow can't be tested — and sandbox latency is the honest test of the
  in-progress states.
- **Restore Purchases is always reachable** (App Store Guideline 3.1.1).
- **Empty offerings are a configuration error**, not a code bug — walk the
  checklist before touching code.
- **Product IDs and entitlement names are permanent** once created. Pick carefully.

## Related, elsewhere

- [`../general-specs-payment/`](../general-specs-payment/README.md) — web checkout,
  plan catalog & limits, billing API, quota enforcement. Its
  [checkout/apple-google-iap.md](../general-specs-payment/checkout/apple-google-iap.md)
  and [ui/restore-purchases.md](../general-specs-payment/ui/restore-purchases.md)
  are the store-side pages of that spec.
- [`../privacy-page/`](../privacy-page/README.md) — carries a copy of
  `non-tracking-purchases/` under `store-label/`; that copy follows this one,
  never the reverse.
