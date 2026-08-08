# SYNC-003 — Durable, device-only credentials (stay signed in)

- **Status:** Adopted
- **Group:** Sync
- **Applies to:** Every app that holds a session token and/or a client-side
  encryption key to talk to a sync backend.
- **Last updated:** 2026-06-17

## Feature

After a successful sign-in the app **MUST** persist its sync credentials on the
device so the user **stays signed in** across app launches and reboots, and
**MUST** keep those secrets **on this device only**. The user re-authenticates
only when they explicitly sign out (or a token is revoked server-side).

## Requirement

1. The session token and any client-held **vault/encryption key MUST** be stored
   in the platform secure store (iOS Keychain / Android Keystore), never in
   plaintext app storage, and the encryption key **MUST NOT** be derivable from
   anything on disk outside the secure store. See
   [SEC-004](../security/SEC-004-transport-and-at-rest.md).
2. Credentials **MUST** survive app relaunch and device reboot. They **MUST** be
   readable on a **cold/background launch after a reboot** — i.e. accessible
   after the first device unlock, not only while the app is foregrounded — so
   launch sync and the periodic cadence ([SYNC-002](SYNC-002-background-sync-cadence.md))
   can run. (iOS: `AFTER_FIRST_UNLOCK`-class accessibility.)
3. Secrets **MUST** be pinned to the device: **not** synced to cloud keychains,
   **not** included in device backups, and **not** migrated to a new device
   (iOS: `…_THIS_DEVICE_ONLY`). A restored/cloned device **MUST** start signed
   out, not inherit another device's vault key.
4. Only an explicit **sign-out** (or server-side revocation) clears the
   credentials. The app **MUST NOT** silently drop the session on a transient
   network/401 blip; token lifecycle/rotation follows
   [AUTH-003](../authentication/AUTH-003-session-token-lifecycle.md).
5. The in-flight web→app handoff secret (e.g. PKCE verifier) **MUST** use the
   same device-only secure storage while pending; see
   [AUTH-006](../authentication/AUTH-006-secure-web-to-app-handoff.md).

## Rationale

"Keep me logged in" is table stakes; users will not re-sign-in on every launch.
But a sync key is exactly the kind of secret that must never ride to another
device via iCloud Keychain or a backup, both for privacy and because a leaked
vault key defeats end-to-end encryption. Device-only, after-first-unlock storage
is the setting that delivers both durability and containment.

## Acceptance criteria

- [ ] After sign-in, force-quitting and relaunching the app stays signed in.
- [ ] After a device reboot, the app launches signed in and can sync before any manual unlock of the app.
- [ ] The token/key are absent from any plaintext storage and from an unencrypted backup.
- [ ] Restoring the device's backup onto a *different* device does not carry the session/vault key.
- [ ] A transient 401/network error does not sign the user out.
- [ ] Explicit sign-out clears token and key from the secure store.

## Implementation notes

- **Tick:** `secure.ts` stores `tick.session`, `tick.encKey`, and `tick.handoff` via `expo-secure-store` with `keychainAccessible: AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY`; `store.bootstrap` restores them on launch; only `logOut`/`deleteAccount` call `clearSecure`. No auto-clear on 401.
