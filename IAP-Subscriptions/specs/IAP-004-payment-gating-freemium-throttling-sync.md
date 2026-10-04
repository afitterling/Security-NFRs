# IAP-004 — Payment gating: freemium, feature throttling & paid iCloud sync

- **Status:** Proposed
- **Group:** IAP & Subscriptions
- **Applies to:** Every app with a free tier and a paid upgrade (subscription or
  one-time unlock), whether bought through the App Store, Play or web checkout. Part
  D applies to every app that sells **iCloud sync** as part of the upgrade.
- **Last updated:** 2026-10-03

## Feature

Every paid app ships a **Free tier that is useful on its own** and a **Pro tier
that adds capabilities**. The gates between the two follow five rules:

- They are declared once, as data.
- They are enforced where the cost lives.
- They explain themselves at the moment they bite.
- They never hold the user's existing data hostage.
- iCloud sync sold as part of Pro follows extra rules, because the platform mirror
  is decided when the store opens and cannot be flipped like a boolean.

## Terms

| Term | Meaning | Example |
|---|---|---|
| **Entitlement** | What the store says the user owns ([IAP-003](IAP-003-revenuecat-integration-baseline.md) §3). | `pro` |
| **Capability** | What the app lets the user do. | `sync.icloud`, `export.pdf`, `ai.scan` |
| **Limit** | A number attached to a capability for a tier. | `lists.max = 3`, `ai.scan.perMonth = 5` |
| **Feature gate** | Free has the capability or does not. | PDF export Pro-only |
| **Quantity limit** | A cap on how many of something *exist*. | 3 lists, 50 journal entries |
| **Rate limit** | A quota per period, with a reset. | 5 AI scans per month |
| **Sync gate** | Cross-device sync is a capability. | iCloud sync is Pro |

## Requirement

### A — Model and enforcement

1. **One capability map.** Entitlements map to capabilities and limits in **one**
   declaration, read by every gate, the paywall comparison table and (if there is
   one) the server: `plans.ts`, `Tiers.swift`. Code **MUST** ask `can(.syncICloud)`
   / `limit(.lists)`. It **MUST NOT** scatter `isPro` checks, and **MUST NOT** ask
   about product ids. Adding or changing a tier is a data change.
2. **Free is genuinely useful.** The core job of the app **MUST** work with no
   purchase, no account and no network. A first-time user **SHOULD** be able to
   get through a typical first week without reaching a limit. A paywall **MUST NOT**
   come before the first use of the core feature, and **MUST** always be
   dismissible.
3. **Limits are disclosed before purchase.** Every limit and every Pro capability
   **MUST** appear in the in-app comparison and in the store description
   (App Review 3.1.2 requires a clear description of what a subscription gives).
   There are no hidden caps in Pro either. If Pro has a fair-use limit, it is stated.
4. **The entitlement comes from the store.** Gates read the verified store or
   RevenueCat entitlement ([IAP-003](IAP-003-revenuecat-integration-baseline.md)
   §11, [`storekit-paywall-gating/`](../storekit-paywall-gating/README.md) NFR-1),
   resolved at launch from the on-device state without waiting on the network
   ([REL-003](../../general-specs/reliability/REL-003-non-blocking-app-boot.md)).
   An app-written "unlocked" flag is never a source.
5. **Enforcement strength follows cost.** Each capability **MUST** be classified in
   the map:
   - **Class L (no cost to us):** local features, and sync into the user's own
     iCloud private database (stored against the user's iCloud quota). A client
     gate on a verified entitlement is sufficient.
   - **Class S (costs us money per use or per month):** AI/LLM calls, our server
     storage, server-side sync, email/SMS, third-party API calls. These **MUST** be
     enforced **server-side**, against the webhook-derived entitlement, with counters
     kept on the server. A client-side count is display only. A breach returns the
     structured `402` body from
     [`general-specs-payment/plans/limits.md`](../../general-specs-payment/plans/limits.md)
     (`metric`, `used`, `limit`, `plan`, `resetsAt`, `upgradeUrl`).
6. **Client gates are layered.** A disabled control is cosmetic. A Class L gate
   **MUST** hold in three places: in the UI, in every mutating action handler, and in
   the feature itself. The gated data **MUST** have exactly one guarded write path
   ([gating-enforcement](../storekit-paywall-gating/gating-enforcement.md)).
7. **No runtime override in App Store builds.** Tiers are tested with a StoreKit
   configuration file, sandbox, or an injected capability map in unit tests. A
   compile-time "pro build" flag for internal installs **MAY** exist only if it is
   excluded from the App Store configuration, and a build check **MUST** assert that.

### B — Throttling behaviour

8. **A quantity limit counts what exists, and blocks only creation.** Deleting an
   item frees room. Reaching the limit **MUST** block only *creating* more. Reading,
   editing, deleting, searching and exporting existing items are never throttled.
9. **A rate limit has a stated period and reset.** The quota **MUST** show its
   period and the next reset ("2 of 5 scans left, resets 1 Nov"). The server returns
   `resetsAt` as an instant, and the client renders it in local time. Only a
   **successful** use consumes quota, so a failed AI call costs the user nothing.
   Consumption **MUST** be idempotent per request id, so a retry is not counted
   twice.
10. **Warn early, block before the effort.** The remaining quota **SHOULD** become
    visible when ≤ 20 % remains. Where the app can know in advance, the gate **MUST**
    trigger *before* the user invests effort, for example when opening the composer
    rather than on Save. If the block happens at commit time, the user's input
    **MUST** be preserved, and it **MUST** be committable after upgrading without
    retyping.
11. **Upgrade in context.** The gate prompt **MUST** say which limit was hit and what
    Pro changes, quote the store's price
    ([IAP-002](IAP-002-price-display-fidelity.md) §7), and offer Purchase **and**
    Restore. Once the entitlement is active
    ([IAP-001](IAP-001-purchase-progress-feedback.md) §4), the blocked action
    **SHOULD** resume on its own.
12. **No dark patterns.** The app **MUST NOT** use fake countdowns or scarcity.
    It **MUST NOT** slow down or degrade free features to push an upgrade. It
    **MUST NOT** show the same gate prompt more than once per session unless the
    user triggers that gate again. Dismissing is always possible.
13. **Lowering a limit is not retroactive.** If a Free limit is reduced, existing
    users keep what they already have. The new limit applies only to future
    creation (rule 8).

### C — Lapse, refund and downgrade

14. **Data is never held hostage.** When Pro ends (expiry, refund, revocation,
    family-sharing removal), everything created under Pro **MUST** stay readable,
    deletable and exportable, and **SHOULD** stay editable. Only creating beyond the
    Free limits is blocked. Export and account deletion
    ([PRIV-002](../../general-specs/privacy/PRIV-002-gdpr-dsgvo-user-rights.md),
    [SYNC-005](../../general-specs/sync/SYNC-005-local-store-source-of-truth.md) §16)
    are never gated.
15. **Lapse is stated once, plainly.** The first launch after a lapse **MUST** say
    what changed ("Pro ended on 2 Oct: sync is paused; your 7 lists are kept, new
    lists need Pro"). It says this once, not on every launch.
16. **Re-subscribing restores everything.** Buying again or restoring **MUST**
    return the full capability with no re-setup, migration or data loss.
17. **Billing grace is honoured.** Store billing grace periods **SHOULD** be enabled
    (App Store Connect "Billing Grace Period"). During grace and billing retry, access
    continues ([IAP-003](IAP-003-revenuecat-integration-baseline.md) §16).

### D — iCloud sync as a Pro capability

These rules extend [SYNC-005](../../general-specs/sync/SYNC-005-local-store-source-of-truth.md)
for apps where `sync.icloud` is gated.

18. **Same store, two configurations.** Free opens the local store with mirroring
    **off**. Pro opens the **same store file** with private-database mirroring
    **on**. There is no second store, no copy, and no export/import between modes.
    Switching in either direction **MUST NOT** lose or duplicate a record. In
    SwiftData terms, the `ModelConfiguration` URL is identical, and only
    `cloudKitDatabase: .none` vs `.private(id)` differs.
19. **The mode is decided at launch from local truth.** The container mode is chosen
    at launch from the on-device verified entitlement (rule 4), never after a network
    round-trip. A Pro user offline still opens in sync mode, and sync resumes when the
    network returns.
20. **Buying turns sync on without reinstalling.** After a purchase, sync **SHOULD**
    start within the session, by rebuilding the container at a safe point (pending
    changes saved, views re-bound to the new container). If the app requires a
    relaunch instead, it **MUST** say so explicitly. It **MUST NOT** silently wait
    for the next cold start.
21. **Lapse pauses sync and never deletes.** On lapse, the device reopens
    local-only, with its local data complete. The copy in the user's private iCloud
    database belongs to the user, and the app **MUST NOT** delete it, except through
    an explicit "Delete iCloud copy" action. On re-subscribe, mirroring resumes and
    the platform merges per record (SYNC-005 §15). The lapse notice (rule 15)
    **MUST** say that the user's devices no longer stay in step.
22. **Don't sell what can't work.** Before a sale where sync is a stated reason to
    buy, the app **MUST** check the iCloud account status (`CKContainer.accountStatus`).
    If there is no account, or it is restricted, the paywall **MUST** say that sync
    needs iCloud and how to turn it on. After purchase, a missing account shows as a
    sync status ("Sign in to iCloud in Settings to start syncing"). For Pro users this
    is the one sign-in hint permitted beyond SYNC-005 §2. Free users still see none.
23. **Sync status is visible to Pro users.** The app shows Synced (with last-synced
    time), Syncing, Paused (Pro ended), Unavailable (no iCloud account), iCloud full
    (`quotaExceeded`), and Error
    ([SYNC-004](../../general-specs/sync/SYNC-004-sync-status-visibility.md)). A full
    iCloud quota is a user-facing state, not a silent stall.
24. **The App Store account and the iCloud account are separate.** The entitlement
    follows the App Store Apple ID. Sync goes to the iCloud account signed in on the
    device. The app **MUST NOT** try to reconcile the two, and **MUST** document the
    case in its support FAQ ([PAGE-003](../../general-specs/pages/PAGE-003-support-route.md)).
    If the product is family-shareable, each family member syncs their **own**
    private data. Nothing is shared between members.
25. **The schema is in production before the first sync build ships.** The CloudKit
    production schema **MUST** be deployed before the first build that can turn on
    Pro sync. From then on, SYNC-005 §13 (additive changes only) applies. Free users
    never depend on it.
26. **Sync via our own server is Class S.** If "sync" means our backend rather than
    the user's iCloud, it costs us storage every month. It is then gated server-side
    (rule 5), and the server **MUST** keep the lapsed user's data for a stated
    retention period before deletion. That period is disclosed on the paywall and
    on `/privacy` ([PRIV-001](../../general-specs/privacy/PRIV-001-data-minimization-and-retention.md)).

## Rationale

Freemium fails in two opposite directions, and both cost revenue.

- **Too leaky.** The gate is only a disabled button, or the server trusts a client
  count, so the costly features (AI, server storage) are effectively free. That
  failure grows with usage.
- **Too hostile.** A paywall before first use, data locked on lapse, a quota with no
  reset date, or a gate that throws away what the user typed. Each turns a would-be
  buyer into a one-star review and, on lapse, a GDPR complaint.

"Enforcement follows cost" settles the first: be strict exactly where we pay, and
lean where we don't. "Throttle creation, never access" settles the second. A user
who sees their data intact and a clear "new lists need Pro" is someone we can sell
to. A user whose data is locked is an adversary.

iCloud sync gets its own part because it is the gate most often built wrong. A
platform mirror is fixed when the container opens, so a naive `if isPro` around
"sync" does nothing, or it opens a second store and forks the data. Treating the
mode as a launch-time configuration of **one** store file, and treating lapse as
pause rather than delete, keeps the sync gate as safe as any boolean gate.

## Acceptance criteria

- [ ] All gates, the paywall comparison and the server read one capability map. Grep finds no product id and no bare `isPro` in feature code.
- [ ] A fresh install with no account and no network can complete the core job.
- [ ] Every Free limit appears in the in-app comparison and in the store description.
- [ ] Every Class S capability is refused by the server for a Free user, even with a tampered client. The `402` body carries `metric`, `used`, `limit` and `resetsAt`.
- [ ] Each Class L gate holds in the UI, in the action handler and in the feature, and the gated data has one guarded write path.
- [ ] An App Store build has no way to reach Pro without a store entitlement.
- [ ] At a quantity limit, existing items stay readable, editable, deletable and exportable, and deleting one allows creating one.
- [ ] A rate limit shows the remaining quota and the reset date. A failed operation does not consume quota, and a retried request is counted once.
- [ ] Hitting a limit at commit time preserves the input, and after a sandbox purchase the action completes without retyping.
- [ ] The same gate prompt does not reappear in one session without the user triggering it again.
- [ ] After a sandbox subscription expires, all data is still present and exportable, and a one-time notice states what changed.
- [ ] Re-subscribing restores the full capability with no setup.
- [ ] Free → Pro → lapse → Pro on one device: record count unchanged at every step, no duplicates, one store file on disk.
- [ ] A Pro user launching offline opens in sync mode, and a sandbox purchase starts sync in the session or shows an explicit relaunch prompt.
- [ ] After a lapse, the iCloud private database still holds the records. They are removed only by "Delete iCloud copy".
- [ ] With the device signed out of iCloud, the paywall states that sync needs iCloud, and a Pro user sees the "Sign in to iCloud" status. A Free user sees no prompt.
- [ ] Sync status shows each state in rule 23, including iCloud full.
- [ ] The CloudKit production schema was deployed before the first build with Pro sync.

## Implementation notes

The notes below say where each pattern already exists. None of these apps has been
audited against this spec.

- **PingTray** (macOS, StoreKit 2 non-consumable) is the reference for a Class L
  **feature gate** with three layers and a single write path: the default host is
  free, custom hosts are Pro
  ([`storekit-paywall-gating/`](../storekit-paywall-gating/README.md)).
- **Trove / Journal** (`macos/Journal/swift/specs/pro-sync.md`) has a **quantity
  limit**: 50 entries Free, unlimited Pro, as a StoreKit 2 universal non-consumable.
  Sync runs over iCloud KVS and is currently not gated. If it becomes Pro, Part D
  applies, and rule 18 means gating the push/pull, not switching stores.
- **Dont Forget Me** sells Pro as a subscription *because* it syncs through our
  server (rule 26, Class S). The store is the source of truth (`ProStore`), and
  RevenueCat is recorded for server-side verification ([IAP-003](IAP-003-revenuecat-integration-baseline.md)
  §10). Its CloudKit container is opened with an explicit `.none` fallback
  (`DontForgetMeApp.makeStore`). That is the right shape for rule 18 once the mode
  key becomes the entitlement, not just a build setting. **Check:** confirm that the
  `DFM_PRO` compile-time flag (`AppConfig.isPro`) cannot be set in the App Store
  configuration (rule 7).
- **WebhookNotification** has the reference **server-side rate limit** (Class S) in
  `sst/src/lib/ratelimit.ts`, with the `402` contract in
  [`general-specs-payment/plans/limits.md`](../../general-specs-payment/plans/limits.md)
  and [`enforcement/rate-limiting.md`](../../general-specs-payment/enforcement/rate-limiting.md).
- **Rebuilding the container (rule 20), SwiftData:** hold the `ModelContainer` in
  an `@Observable` app model and inject it with `.modelContainer(model.container)`.
  On an entitlement change: `try context.save()`, build a new container on the same
  URL with the other `cloudKitDatabase`, then assign it. Run the
  [SYNC-005](../../general-specs/sync/SYNC-005-local-store-source-of-truth.md) §6
  repair step before views re-read. Never open both configurations on the same file
  at once.
- **Capability map sketch (Swift):**
  ```swift
  enum Capability { case syncICloud, exportPDF, aiScan }
  enum Limit { case lists, aiScanPerMonth }
  struct Tier { let caps: Set<Capability>; let limits: [Limit: Int] }  // Int.max = unlimited
  static let tiers: [String: Tier] = [            // keyed by entitlement id, "" = free
      "":    Tier(caps: [],                       limits: [.lists: 3, .aiScanPerMonth: 0]),
      "pro": Tier(caps: [.syncICloud, .exportPDF, .aiScan], limits: [.lists: .max, .aiScanPerMonth: 50]),
  ]
  ```
