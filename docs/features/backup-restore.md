# Backup & Restore

Mium can periodically ship its control-plane state — the **KMS keystore**, the **IAM** snapshot, and the **Connection Store** — to an S3-compatible target. Backups are leader-only, incremental, opt-in, and run on either a fixed interval, a 5-field UNIX cron expression, or both at once.

**Intentionally not in scope:** Memory, Prompt Library, and Embedding Settings live in NeorunBase. They are restored by restoring NeorunBase itself (see [the NeorunBase backup docs](https://cloudcheflabs.github.io/neorunbase-docs/latest/features/backup-restore/)). Mium's own backup only covers what is local to the Mium master.

## Configuration

The `Backup & Restore` page in the admin UI exposes the full configuration:

- **Enabled** — master switch (default: off).
- **S3 Endpoint, Region, Bucket, Prefix** — destination object store.
- **Access Key / Secret Key** — long-lived static credentials for the destination.
- **Path-style addressing** — required for ShannonStore / MinIO destinations.
- **Interval (minutes)** — how often the scheduled backup runs.
- **Cron** — optional 5-field UNIX cron expression (e.g. `0 2 * * *` daily at 02:00). Coexists with the interval rule — both can be set, whichever comes due first fires the backup.
- **Retention (days)** — how long the visible history is kept.

The same page has a **Backup Now** button and a **Restore** button on each row of the available-backups list.

### Cron vs. Interval

The two automatic rules are independent and share the same tick loop on the leader master:

- **Interval** is a relative "at least N minutes since the last successful backup" timer.
- **Cron** is wall-clock based, evaluated in the leader's local time zone.

The first sighting after the cron is set, or after a leader handoff, **arms from "now"** — Mium does not back-fire missed cron times on startup. An invalid cron is rejected with HTTP 400 at config-save time rather than being silently skipped at tick time.

## What gets backed up

Each backup captures three stores under one shared backup id, so a single restore reassembles a consistent point in time:

| Store | Purpose |
|---|---|
| `kms`         | KMS keystore (all DEKs/KEKs) |
| `iam`         | Users, groups, policies, access keys |
| `connections` | Encrypted external-connection definitions |

Each store is content-hashed (SHA-256). If a store's bytes are unchanged since the last backup, it is **not re-uploaded** — the new backup's manifest simply points at the backup id that already holds it. A backup taken with no Mium activity uploads only a tiny manifest.

## Restore

The available-backups list shows every backup id present at the configured prefix, newest first. **Restore** is destructive — it overwrites the live KMS, IAM, and Connection stores with whichever snapshot the chosen backup id points to (possibly an older backup id, per the incremental layout). The admin UI confirms before proceeding.

Restore is leader-only and gated by the `SYSTEM:MANAGE_BACKUP` IAM action; followers receive the broadcast state through the existing sync protocols.

## REST Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET`  | `/api/backup/config`  | Read current config (secret never echoed) |
| `POST` | `/api/backup/config`  | Update; sticky secret (only overwrites on non-empty); `MANAGE_BACKUP` |
| `POST` | `/api/backup/run`     | Trigger an immediate backup; `MANAGE_BACKUP` |
| `GET`  | `/api/backup/history` | Last 100 history entries |
| `GET`  | `/api/backup/list`    | Backup ids in S3, newest first |
| `POST` | `/api/backup/restore` | Restore from `{backupId}`; `MANAGE_BACKUP` |

All mutating routes are leader-only and forwarded automatically by the admin HTTP server.
