# UI — Restore Purchases (subscription restore)

A one-tap way for a returning customer to recover their paid entitlement on a
device that doesn't currently show it — new phone, reinstall, or after signing
out. On iOS a **Restore** mechanism is an App Store Review requirement
(Guideline 3.1.1) for non-consumable / auto-renewing purchases.

> Worked example: **Nilo** (`timer/[pathiosapp]`, Expo RN + `react-native-purchases`).
> The restore *mechanism* already exists; the gap is **discoverability** — it's
> only reachable from the paywall, whose entry buttons say *"Get Nilo Pro"*. A
> returning buyer won't tap "Get Pro" to restore. This adds a dedicated row to
> the settings screen.

## Two restore paths (don't conflate them)

| Path | Mechanism | Fires | Needs the RC SDK? |
|---|---|---|---|
| **Account-based** | server `/entitlement`, fed by the RevenueCat **webhook** setting `user.plan`/`pro` | **automatically** on sign-in & boot | No (works even with no IAP key) |
| **Apple-ID-based** | `Purchases.restorePurchases()` (StoreKit) | **manually**, this button | Yes — *is* RevenueCat |

Signing into the account already restores Pro with no button. The **button**
covers the Apple-ID purchase that the current session doesn't reflect (bought
signed-out, broken account/webhook linkage, reinstall before sign-in). See
[../checkout/revenuecat.md](../checkout/revenuecat.md) and
[../checkout/apple-google-iap.md](../checkout/apple-google-iap.md).

## Where

A **"Restore Purchases"** row in the settings/Account screen's subscription area,
shown to **non-Pro** users (the paywall keeps its own restore button too). Nilo:
`[pathiosapp]/src/screens/AccountScreen.tsx`, near the "Get Nilo Pro" section
(`<PaywallSheet>` is already rendered there).

## What it calls (RC-only — no backend change)

Reuse the existing store action — do **not** add new purchase logic:

```
restorePro(): Promise<boolean>      // store.tsx — wraps restore(), setPro(ok), reconcile
  -> restore()                      // purchases.ts — isProInfo(await P().restorePurchases())
        guarded by purchasesAvailable()  // false when REVENUECAT_IOS_KEY is empty
```

On success the entitlement listener flips `isPro`, so the whole screen reactively
updates to the Pro state (sync unlocks, "Get Nilo Pro" → Pro) underneath the alert.

## Three outcomes, three messages

Mirror `PaywallSheet.doRestore()` but, since the settings screen stays open
(no sheet to auto-dismiss), confirm with a native `Alert.alert` (the screen's
existing feedback idiom):

| Outcome | Condition | Alert |
|---|---|---|
| ✅ Restored | `restorePro()` → `true` | **"Nilo Pro restored"** — "Your subscription is active on this device again." |
| ➖ Nothing found | `restorePro()` → `false` | **"No purchase found"** — "We couldn't find a Nilo Pro purchase on this Apple ID." (neutral, **not** an error) |
| ⚠️ Failed | call throws | **"Couldn't restore"** — "Something went wrong. Please try again." |

```tsx
const doRestore = async () => {
  setBusy(true);
  try {
    const ok = await restorePro();
    Alert.alert(
      ok ? 'Nilo Pro restored' : 'No purchase found',
      ok ? 'Your subscription is active on this device again.'
         : "We couldn't find a Nilo Pro purchase on this Apple ID.",
    );
  } catch {
    Alert.alert('Couldn’t restore', 'Something went wrong. Please try again.');
  } finally {
    setBusy(false);
  }
};
// Row: <Pressable onPress={doRestore} disabled={busy}> {busy ? 'Restoring…' : 'Restore Purchases'} </Pressable>
```

Reuse PaywallSheet's `restore`/`restoreTxt` styles for visual consistency.

## Gotchas / testing

- **Dev build can't test it.** With `REVENUECAT_IOS_KEY` unset (Nilo's `nilodev`
  build), `purchasesAvailable()` is false → `restore()` always returns `false`
  ("nothing found"). Test on a **prod/TestFlight** build with a sandbox Apple ID.
- **`restorePurchases()` may prompt** the StoreKit / Apple-ID sign-in sheet —
  expected; it's how Apple authenticates the restore.
- **`false` ≠ error.** Keep the neutral "nothing found" copy distinct from the
  thrown-error copy, or users think a real restore failed.
- **RC-only is intentional.** If a user is entitled **server-side** but StoreKit
  returns `false`, this reports "nothing found" — acceptable because sign-in
  already restores that case. Optional later hardening: on `false` + signed-in,
  re-pull `/entitlement` as a fallback (deferred).
- **iOS only.** Android/web restore differ (out of scope here).
