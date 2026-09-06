# Non-Tracking Purchases — ATT vs. Privacy Label (Guideline 5.1.2(i))

**The NFR:** an app's runtime tracking behavior and its App Store privacy label
must agree. RevenueCat/StoreKit make this easy to get wrong, because collecting
*purchase data* is not the same as *tracking* — but a mislabeled purchase looks
like tracking to App Review and gets the app rejected under **5.1.2(i)**.

Reusable, parameterized. **Nilo** (bundle `tech.sp33c.nilo`, RevenueCat entitlement
`pro`, Expo/React Native) is the worked example — substitute your own bundle and
RevenueCat setup. Complements `../../IAP-Subscriptions/app-store-iap-setup/`,
`../../IAP-Subscriptions/revenuecat-integration/`, and
`../../IAP-Subscriptions/storekit-paywall-gating/`.

> Status: **proven, painful**. This was a rejection on Nilo submission
> `ffe89067-…` (v1.4.4), then a *repeat* rejection because the stale ATT string
> was still in the shipped binary after we thought we'd removed it (see the
> verification trap below).

## Apple's definitions (the distinction that matters)

- **Collecting** Purchase History = you receive what the user bought. Fine, and
  unavoidable with any IAP. Declared under *Data Collected → Purchases*.
- **Tracking** = linking that data with third-party data **for advertising**, or
  sharing it with a **data broker**. Only tracking triggers the ATT prompt.
- A subscription app that uses RevenueCat purely for **entitlements + first-party
  attribution does NOT track.** It only tracks if you enable IDFA/advertising-id
  collection (RevenueCat: `collectDeviceIdentifiers()`).

**The rejection trap:** the privacy label lists Purchase History under *"Used to
Track You"*, but the app never shows the ATT prompt (or the shipped binary lacks
the ATT wiring). Label says "tracks", app doesn't ask → 5.1.2(i) rejection.

## Decision: go **no-tracking** (recommended default)

For most utility/subscription apps, IDFA attribution is worth ~nothing. Declaring
no tracking is Apple's own first-listed resolution and removes the ATT prompt
dependency entirely. Two things must then be true, and BOTH are load-bearing:

1. **Label:** Purchase History is *Collected* (linked to identity for App
   Functionality is fine) but **NOT** under "Used to Track You".
2. **Binary:** the app collects **no advertising identifier** and ships **no**
   `NSUserTrackingUsageDescription`.

If either is out of sync, review bounces you — and Apple checks the *binary*, not
your intent. A leftover `NSUserTrackingUsageDescription` in the shipped Info.plist
alone triggers: *"Your app contains NSUserTrackingUsageDescription… update your
privacy response to indicate tracking, or upload a new build."*

## Removing tracking cleanly (Expo / RN + RevenueCat worked example)

To match a no-tracking label, remove every ATT/IDFA surface:

1. Delete the ATT request helper (`src/tracking.ts`) and its call site
   (`requestTrackingOnce()` in `App.tsx`).
2. Remove RevenueCat device-identifier collection —
   `collectDeviceIdentifiers()` / any `enableDeviceIdentifierCollection()` wrapper
   in `src/purchases.ts`. (RevenueCat collects **no** IDFA unless you call this.)
3. Remove the ATT config plugin (`expo-tracking-transparency`) from `app.json`
   **and** `npm uninstall expo-tracking-transparency` (so the framework isn't linked).
4. **Verify the *generated* Info.plist**, don't assume the plugin removal is enough:

   ```
   APP_ENV=prod npx expo config --type introspect | grep -c NSUserTracking   # must be 0
   ```

### ⚠️ The verification trap that caused a *repeat* rejection

`expo config --introspect` **merges onto the existing committed `ios/` native
project** — it does *not* simulate `prebuild --clean`. So after removing the
plugin, introspect can *still* report `NSUserTrackingUsageDescription` because the
stale key is sitting in the committed `ios/Nilo/Info.plist` from a prior build.

Removing the config plugin only stops the key from being *re-added*; it does not
delete a key already present in the native project. Result: you believe it's gone,
re-upload, and Apple flags the **same** ATT string again.

**Fix / confirm both layers:**
- Config layer: plugin gone from `app.json`, package uninstalled.
- Native layer: `NSUserTrackingUsageDescription` absent from the shipped Info.plist
  — either delete it manually, or let `prebuild --clean` regenerate a clean plist
  (which it does once the plugin is gone). Re-run the introspect grep → `0`.

## Keeping tracking (the other valid path)

If you genuinely want IDFA attribution: keep the label as-is, keep the ATT prompt,
and in Review Notes state **exactly where/when the prompt appears** (e.g. "~1s
after first launch"). Risk: the reviewer must actually *see and accept* the prompt
on their device, or it reads as "doesn't ask". No-tracking avoids this entirely.

## Checklist
- [ ] Decide tracking posture **before** filling the privacy label.
- [ ] No-tracking: no `collectDeviceIdentifiers()`, no ATT plugin/package, no
      `NSUserTrackingUsageDescription` in the **shipped** Info.plist.
- [ ] `expo config --introspect | grep -c NSUserTracking` → `0` (and confirm the
      committed `ios/` plist too — introspect merges it).
- [ ] Privacy label: Purchase History **not** under "Used to Track You".
- [ ] Upload a **new binary** — a label-only change won't clear a binary that still
      carries the ATT string.

See also `../../IAP-Subscriptions/revenuecat-integration/attribution-and-webhook-sync.md` for what
RevenueCat actually collects.
