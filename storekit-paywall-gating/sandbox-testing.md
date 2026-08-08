# Repeatable sandbox test of the paywall (end to end)

Goal: exercise the real StoreKit flow — locked → purchase → unlocked → restore — with a sandbox
Apple Account, no debug shortcuts. Do this every release that touches the paywall.

## Prerequisites
- A **sandbox tester** (App Store Connect → Users and Access → Sandbox → Test Accounts). Save
  its email; keep its password in a password manager. One non-consumable can be re-tested by
  clearing its purchase history.
- IAP product `state = READY_TO_SUBMIT` and attached to the build (verify via the API in
  `../app-store-iap-setup/app-store-connect-api.md`).
- Most faithful surface: a **TestFlight** build. A locally ad-hoc-signed build may fail to load
  the product (`Product.products(for:)` empty → Unlock button disabled). See gotchas.

## Get to the locked state (from scratch)
1. App Store Connect → Sandbox → the tester → **Clear Purchase History**.
2. Quit the app.
3. Wipe local app state so no cached transaction lingers:
   ```sh
   defaults delete <bundle-id>
   rm -f  ~/Library/Preferences/<bundle-id>.plist
   rm -rf ~/Library/Containers/<bundle-id>
   rm -rf ~/Library/Caches/<bundle-id>
   killall cfprefsd
   ```
4. Relaunch. Premium UI should now show **locked** (Unlock + price visible, fields disabled).

## Test the purchase
5. Trigger Unlock → the StoreKit purchase sheet appears → sign in with the **sandbox** tester.
6. Complete purchase → app unlocks; premium feature activates immediately (entitlement listener
   fires `refreshEntitlement`).

## Test restore
7. Wipe local state again (steps 2–4) WITHOUT clearing purchase history → relaunch → locked.
8. Trigger **Restore** → `AppStore.sync()` → entitlement returns → unlocked. (If nothing to
   restore, the app reports it.)

## What to assert
- Locked: cannot add/edit/remove premium items; premium feature inactive; only the free tier
  runs.
- Purchase: unlock is immediate and survives relaunch.
- Restore: re-grants on a clean install for the same Apple Account.
- Refund path (optional): App Store Connect refund / `revocationDate` set → app re-locks on
  next entitlement refresh.

## Cannot be automated
The StoreKit purchase sheet is a secure system modal requiring the sandbox password — drive it
by hand. A harness can only set the stage (clear/relaunch) and assert the before/after state.
