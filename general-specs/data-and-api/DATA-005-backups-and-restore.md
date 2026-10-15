# DATA-005 — Backups & restore

- **Status:** Adopted
- **Group:** Data & API
- **Applies to:** Every app with a `production` stage that stores data in DynamoDB.
- **Last updated:** 2026-10-15

## Requirement

1. Every production DynamoDB table **MUST** have point-in-time recovery (PITR)
   enabled with the full 35-day recovery period.
2. Every production DynamoDB table **MUST** be covered by a daily snapshot that
   is kept for **14 days** (rolling window: each recovery point is deleted 14
   days after it was taken).
3. The daily snapshots **MUST** be owned centrally by the AWS-Admin app
   `aws-admin-backups` (`Admin-Home/AWS-Admin/backups`). An app is covered by
   registering its SST app name there. Apps **MUST NOT** define their own AWS
   Backup plans or vaults.
4. Tables **MUST** keep the SST naming `<app>-production-<Name>Table-<suffix>`
   (no custom `name:` on `sst.aws.Dynamo`), because the central selection
   matches the `<app>-production-*` prefix.
5. Dev, preview and per-person stages **MUST NOT** be backed up. They use
   `removal: remove` and would only leave orphaned recovery points behind.
6. A backup job that does not finish as `COMPLETED` **MUST** alert by email.
7. A restore **MUST** go to a new table. Restoring over the live table name is
   not allowed.
8. A restore from a daily snapshot **SHOULD** be tested once per quarter per app
   and the date noted in the app's `TODO.md` or release notes.
9. Production tables **SHOULD** set `deletionProtection: true` so that a
   deleted stack or a bad `sst remove` cannot drop the table itself.

## Rationale

PITR covers bad writes inside the last 35 days at second granularity, but its
history goes with the table. The daily snapshots live in a separate vault and
survive a deleted table or stack. Owning the plan in one place keeps the
schedule, retention and alerting identical across apps, and makes coverage
visible in one file instead of being re-implemented, slightly differently, in
every `sst.config.ts`. Snapshots of small tables cost cents per month, so all
tables of a registered app are included, ephemeral ones too.

## Acceptance criteria

- [ ] `aws dynamodb describe-continuous-backups` shows PITR `ENABLED`, 35 days,
      for every `<app>-production-*` table.
- [ ] The app's SST name is listed in `APPS` in
      `Admin-Home/AWS-Admin/backups/sst.config.ts`, and that app is deployed.
- [ ] The vault `dynamodb-daily-14d` holds a recovery point per table from the
      last 24 hours, and none older than 14 days.
- [ ] The app's own `sst.config.ts` contains no `aws.backup.*` resources.
- [ ] The SNS subscription for `aws-admin-backup-alerts` is confirmed.
- [ ] A restore test to a new table succeeded within the last quarter.

## Implementation notes

- **Central plan:** `aws-admin-backups` — plan and vault `dynamodb-daily-14d`,
  daily at 01:00 UTC, `deleteAfter: 14`, one selection per app on
  `arn:aws:dynamodb:<region>:<account>:table/<app>-production-*`, alerts via
  SNS `aws-admin-backup-alerts`. Restore commands are in its README.
- **WebHookPush (`webhook`):** PITR on all 9 production tables. Registered.
  Its own plan `WebhookDailyBackup` (14 d, 6 tables) stays until the central
  vault holds a full 14-day window, then it is removed.
- **TimeWarp (`timewarp-sst`):** PITR on all 4 production tables. Registered.
  Its own plan `TimeWarpBackupPlan` (10 d, 2 tables) is removed on the same
  schedule.
