# Gating enforcement — three layers + single write path

A disabled button is cosmetic. To actually gate a paid feature, enforce at every layer and
prove there is exactly one write path, all guarded.

## The entitlement source (Store.swift)
```swift
func refreshEntitlement() async {
    var unlocked = false
    for await result in Transaction.currentEntitlements {
        if case .verified(let t) = result,
           t.productID == Self.productID,
           t.revocationDate == nil {        // refunded/revoked => not unlocked
            unlocked = true
        }
    }
    // NOTHING else. No UserDefaults flag, no #if DEBUG override.
    await MainActor.run { self.isUnlocked = unlocked; self.onChange?() }
}
```
- `start()` also listens to `Transaction.updates` and re-runs this on any change, and calls it
  once at launch. `onChange` drives all UI/monitor refreshes.

## Layer 1 — UI reflects lock state
`HostsWindowController.refresh()` reads `store.isUnlocked` and:
- shows the upsell ("Unlock — $price" + Restore) only when locked (`isHidden = unlocked`);
- sets `hostField/portField/addButton/editButton/removeButton .isEnabled = unlocked`.

## Layer 2 — action handlers guard (defense in depth)
The disabled controls are not trusted. Each action that can mutate premium data starts with:
```swift
@objc private func addTapped()    { guard store.isUnlocked else { return }; … }
@objc private func removeTapped() { guard store.isUnlocked else { return }; … }
@objc private func editSelected() { guard store.isUnlocked else { return }; … }
```

## Layer 3 — the feature itself is gated
Even if stale premium data exists on disk, it does nothing when locked. In `main.swift`:
```swift
private func rebuildCustomMonitors() {
    customMonitors.forEach { $0.stop() }; customMonitors = []
    guard store.isUnlocked else { return }     // locked => no custom monitoring at all
    for host in hostStore.load() { … start … }
}
```

## The single-write-path rule (how to verify)
Premium data must have exactly ONE persistence path, reachable only through guarded actions.
For PingTray:
`addTapped/removeTapped → commit() → appDelegate.hostsWindowDidEdit() → hostStore.save()`.
`commit()` has no other callers. Verify with grep before shipping:
```
grep -rn "hostsWindowDidEdit\|\.save(" Sources/   # one writer
grep -rn "guard store.isUnlocked" Sources/        # every mutating action guarded
```
If a new code path can write premium data without passing a guard, the gate is broken.

## Note on saved-data-while-locked
We chose to KEEP the user's premium data dormant when locked (it resumes on re-unlock) rather
than delete it. That is a deliberate product choice; if a spec requires "locked = no data
visible", clear/hide it in the locked branch explicitly.
