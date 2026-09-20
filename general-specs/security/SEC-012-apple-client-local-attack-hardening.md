# SEC-012 — Apple client hardening against local attacks (the Swift run-through)

- **Status:** Proposed
- **Group:** Security
- **Applies to:** Every app that ships an Apple-platform client — Swift/SwiftUI
  and Mac Catalyst targets directly, and Expo/React Native iOS builds through
  their native equivalents. This is the per-app, per-release **run-through**:
  every clause is something a person can check on a real build. Server-side
  protection is [SEC-004](SEC-004-transport-and-at-rest.md); credential storage
  overlaps [SYNC-003](../sync/SYNC-003-durable-credentials.md), which this spec
  extends from "stays signed in" to "and cannot be lifted off the device".
- **Last updated:** 2026-09-20

## Threat model — what "local" means here

The attacker **has the hardware or a foothold on it**: a stolen or borrowed
device, a device found locked, a shoulder over the screen, a malicious or merely
careless second app on the same device, a Mac user account with access to the
user's files, or an unencrypted backup on a laptop. The attacker is assumed to
be able to read anything the app leaves in its container in the clear, read
anything shared through a group container or the pasteboard, launch the app,
follow a link into it, and — on a jailbroken/rooted device — attach to it.

**Out of scope here:** a network attacker ([SEC-004](SEC-004-transport-and-at-rest.md)),
a compromised dependency ([SEC-009](SEC-009-dependency-and-build-supply-chain.md)),
and a compromised backend. The question this spec asks is narrower: *if someone
holds this device, what do they get?*

## Requirement

### On-device storage

1. Credentials, tokens, vault/encryption keys, and recovery material **MUST**
   live in the Keychain, never in `UserDefaults`, a plist, a JSON file, SwiftData
   /Core Data, or a log. `UserDefaults` is a plist in the container: readable
   from a backup, readable on a Mac, and readable by anything with the file.
2. Keychain items holding device-bound secrets **MUST** use a
   `…ThisDeviceOnly` accessibility class (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`
   for anything a background launch must read; `…WhenUnlockedThisDeviceOnly` for
   the rest) and **MUST NOT** set `kSecAttrSynchronizable`. `kSecAttrAccessibleAlways`
   (and its `ThisDeviceOnly` variant) **MUST NOT** be used.
3. High-value actions (revealing a secret, exporting data, disabling a safety
   feature) **SHOULD** be gated behind `SecAccessControl` with `.userPresence`,
   and where the item is the credential itself, **SHOULD** use
   `.biometryCurrentSet` so that enrolling a new fingerprint or face invalidates
   it rather than inheriting access.
4. On iOS, the app's local database and any file holding personal data **MUST**
   be created with a Data Protection class of at least
   `.completeUntilFirstUserAuthentication`, explicitly set rather than inherited,
   and the **sidecar files count** — an SQLite/SwiftData store means the `-wal`
   and `-shm` files too. On macOS, where Data Protection classes do not exist,
   the equivalent is the Keychain plus the user's FileVault, and the app **MUST
   NOT** write personal data outside its container to compensate.
5. The Keychain **outlives app deletion** on iOS. The app **MUST** detect first
   launch after a fresh install (a marker in `UserDefaults`, which *is* wiped)
   and purge stale Keychain items, so a reinstalled app on a resold device does
   not resume someone else's session.
6. Sensitive server responses **MUST NOT** be left in `URLCache` on disk; use an
   ephemeral configuration or a memory-only cache for authenticated requests.

### What leaves the device without anyone attacking it

7. Tokens, keys, caches, and derived files **MUST** be excluded from backups
   (`URLResourceValues.isExcludedFromBackup`), so an unencrypted local backup
   does not carry them off the device. This is the same requirement as
   [SYNC-003](../sync/SYNC-003-durable-credentials.md) clause 3, checked from
   the file side.
8. An app that mirrors to CloudKit **MUST** confirm which database each record
   class lands in — private vs. shared vs. public — and **MUST NOT** place
   user-entered content in the public database. A local input that silently
   becomes world-readable is the highest-consequence, lowest-effort mistake on
   this list.
9. Secrets and personal data **MUST NOT** be written to the pasteboard
   implicitly. A deliberate "copy" action **SHOULD** set `.localOnly` and an
   `.expirationDate` so the value does not cross to the user's other devices via
   Universal Clipboard and does not linger.
10. Logging **MUST NOT** emit tokens, keys, or personal data. `Logger` redacts
    dynamic interpolations by default — the real risk is a `.public` annotation
    or an `NSLog` added while debugging and shipped. Diagnostics shown to support
    follow [REL-005](../reliability/REL-005-diagnostics-surface.md) and
    **MUST** be scrubbed at the point of capture, not at the point of display.
11. Personal content **MUST NOT** be indexed into Spotlight/CoreSpotlight, an
    App Intent result, or a widget timeline unless that exposure is intended —
    each of those surfaces renders on the **lock screen** or in a system UI
    outside the app's authentication.

### The screen and the front door

12. Fields carrying credentials **MUST** use secure text entry, and the app
    **MUST** obscure sensitive screens when backgrounded (an overlay on
    `willResignActive`), because the system snapshots the window for the app
    switcher and stores it in the container.
13. Where a screen displays high-value data, the app **SHOULD** react to
    `UIScreen.isCaptured` (recording or mirroring) by masking it.
14. Any state-changing action reachable by a deep link — custom scheme, universal
    link, App Intent, Shortcut, or handoff payload — **MUST NOT** execute on
    arrival; it **MUST** land on a screen that requires a deliberate confirmation
    by an authenticated user. **Any app can register a custom URL scheme**, so
    scheme input is untrusted input. This is the same principle as the inert-GET
    confirm link in [`double-opt-in-auth/`](../../double-opt-in-auth/README.md).
15. Universal links **SHOULD** be preferred over custom schemes for anything
    carrying a token or an identifier, since they are bound to a domain we own.
16. Deserialization of anything read from disk, a shared container, or a pasteboard
    **MUST** use `NSSecureCoding` with an explicit expected class list — never an
    unconstrained top-level unarchive.

### Sharing surfaces and the second app on the device

17. App Group containers and shared Keychain access groups are readable by **every
    app in the group**, including a future one. What goes in a shared container
    **MUST** be limited to what the extension genuinely needs, and the group
    membership **MUST** be reviewed when a new target is added.
18. On macOS, an XPC service **MUST** authenticate its client by **audit token**
    (code-signing requirement check), never by PID — a PID can be reused between
    the check and the use.
19. macOS targets **MUST** ship with App Sandbox and Hardened Runtime enabled and
    with the **minimum** entitlement set; `disable-library-validation`,
    `allow-unsigned-executable-memory`, and `allow-dyld-environment-variables`
    **MUST NOT** be present without a written justification in the app's notes,
    since each one re-opens code injection into a signed process.
20. Code **MUST NOT** be loaded from a user-writable path (plug-in directories,
    `~/Library`, a downloads folder), and helper tools **MUST** be installed
    through the platform mechanism rather than copied into place.

### Build and runtime posture

21. Distribution builds **MUST NOT** carry `get-task-allow`, **MUST NOT** enable
    arbitrary-load ATS exceptions (`NSAllowsArbitraryLoads`), and **MUST NOT**
    contain debug menus, test accounts, seeded credentials, or a local debug
    HTTP/Bonjour listener.
22. No secret **MUST** be embedded in the binary or `Info.plist`. An API key in a
    shipped app is a published key — `strings` is enough. Anything that must stay
    secret belongs behind our backend ([SEC-001](SEC-001-secrets-management.md)).
23. Entitlement-bearing decisions (subscription state, feature access) **MUST**
    be authoritative server-side; client-side gating is UX, not enforcement,
    because on a jailbroken device the client is the attacker's to edit.
24. Jailbreak/tamper detection **MAY** be implemented, but **MUST NOT** be the
    sole control protecting anything, and **MUST NOT** brick the app — it is a
    signal, not a boundary.
25. **Sign-out MUST wipe**: Keychain items, the local store, `URLCache`, cookies,
    `WKWebsiteDataStore`, shared-container copies, and any queued outbound data.
    A web view used for authentication **MUST** use a non-persistent data store.

## The run-through

Do this per app, before a release, in order. It is deliberately a sequence of
*looks at a real build*, not a code review — most findings on this list are
invisible in the source and obvious in the container.

1. **Pull the container.** Simulator: `xcrun simctl get_app_container booted <bundle-id> data`.
   Device: Xcode → Devices and Simulators → *Download Container*. Then walk it:
   every plist, every `.sqlite`/`.store` (plus `-wal`/`-shm`), every cache.
   Anything readable that should not be readable is a finding — clauses 1, 4, 6.
2. **Grep the source for the storage escapes**: `UserDefaults` writes near
   auth code, `kSecAttrAccessible*` values, `kSecAttrSynchronizable`. Clauses 2, 3.
3. **Check what backs up.** Look for `isExcludedFromBackup` on the token/cache
   paths; for a CloudKit app, list every record type and the database it targets.
   Clauses 7, 8.
4. **Read the log stream while exercising sign-in and sync**, and search it for
   the token you just used. Clause 10.
5. **Background the app on a sensitive screen**, then open the app switcher and
   look at the snapshot. Turn on screen recording and look again. Clauses 12, 13.
6. **Fire every deep link by hand** (`xcrun simctl openurl booted "<scheme>://…"`),
   including while signed out and with mutated parameters, and confirm nothing
   changes state without a tap. Clauses 14, 15.
7. **Inspect the shipped entitlements and binary**:
   `codesign -d --entitlements :- <App>.app` (get-task-allow, sandbox, hardened
   runtime, app groups, keychain groups), `plutil -p Info.plist` (ATS, URL
   schemes), `strings` the binary for anything key-shaped. macOS additionally:
   `spctl -a -vvv -t exec <App>.app`. Clauses 17, 19, 21, 22.
8. **Run the sign-out drill**: sign in, sync, sign out, then re-pull the
   container and confirm what remains. Then delete the app, reinstall, and launch
   it — it must come up signed out. Clauses 5, 25.
9. **Record the result** in the app's notes with the date and the build, and
   file every finding as a gap in that app's backlog. A run-through that found
   nothing and was never written down did not happen.

## Rationale

Every other spec in this group defends a server. The Apple client is the one
place where the attacker can hold the machine, and the failures there are not
clever — they are a token in `UserDefaults`, a database that is readable because
nobody set a protection class, a debug `NSLog`, a CloudKit record type pointed at
the public database, and an app-switcher snapshot of the screen that shows the
data. None of these are exploited remotely; all of them are read.

The run-through exists because these findings do not show up in a code review.
The source says `keychain.set(token)` and looks fine; the container shows the
same token in a plist written by a cache layer two years ago. Looking at a real
build is the control.

The two clauses most often argued with are 23 and 24. Client-side entitlement
checks feel like enforcement and are not — on a device the user controls, the
client is not a trust boundary, and treating it as one moves the authoritative
decision to the machine most exposed to the attacker. Conversely, jailbreak
detection feels like a defense and is at best a signal: it loses a race against
the tooling it detects, and an app that hard-fails on it mostly annoys legitimate
users on modified devices.

## Acceptance criteria

- [ ] No token, key, or personal datum is readable in a pulled container.
- [ ] Every Keychain write names an explicit accessibility class; none is
      `Always` or synchronizable.
- [ ] The local store and its `-wal`/`-shm` carry an explicit protection class.
- [ ] A fresh reinstall launches signed out.
- [ ] Token and cache paths are excluded from backup.
- [ ] Every CloudKit record type's target database is listed and none is public.
- [ ] Signing in and syncing produces no log line containing the token.
- [ ] The app-switcher snapshot of a sensitive screen shows the overlay.
- [ ] No deep link changes state without an authenticated confirmation.
- [ ] Shipped entitlements contain no `get-task-allow`, no ATS arbitrary loads,
      and no unjustified hardened-runtime exception.
- [ ] `strings` on the binary reveals no API key or secret.
- [ ] Sign-out leaves nothing recoverable in the container.
- [ ] The run-through is recorded, dated, and tied to a build number.

## Implementation notes

- **Shopping List (SwiftData + CloudKit, iOS + Mac Catalyst):** the highest-value
  clauses here are 4 (SwiftData store protection class, sidecar files included),
  8 (mirroring is to the user's **private** database — verify, don't assume), and
  11 (list content in Spotlight/widgets is lock-screen-visible). There is no
  app-held credential — iCloud is the OS's — so clauses 1–3 mostly fall away,
  which is exactly why the storage clauses matter more.
- **Emergency (Guarding Angel):** the device key *is* the credential, so clause 2
  (`AfterFirstUnlockThisDeviceOnly`, never synchronizable) and clause 5 are
  load-bearing, and clause 14 applies directly to the web→app handoff
  ([AUTH-006](../authentication/AUTH-006-secure-web-to-app-handoff.md)) — the
  handoff link must land on a confirmation, not on an action.
- **Expo clients (OpenCycle, OpenOutdoor, Priorize, WebhookNotification):**
  `expo-secure-store` maps to the Keychain, but the **accessibility option must
  be passed explicitly** — the default is not `ThisDeviceOnly`. `AsyncStorage` is
  unencrypted and is the clause-1 trap on these apps. Clauses 12, 14, 21 and 22
  apply through app config and JS; clause 4 needs a native check on the store
  files the RN layer creates.
- **Mac Catalyst specifics:** Data Protection classes do not apply, sandbox and
  hardened runtime do, and the container is an ordinary folder in the user's
  Library — meaning clause 1 has no fallback on macOS whatsoever.
- **What this spec does not do:** it does not require an obfuscation or RASP
  product. On a device the attacker owns, those raise cost; they do not create a
  boundary, and the clauses above are what actually determine what is lost.
