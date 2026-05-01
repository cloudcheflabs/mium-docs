# Retention & Sweeps

Mium accumulates data on three tiers: bootstrap RocksDB on each Master, NeorunBase tables, and the S3 export bucket. Without retention, chat history grows forever, the prompt library bloats, and exported files pile up in S3. The leader runs a periodic sweep that enforces retention rules — but **every TTL ships disabled by default**, because operators in regulated environments often need to pin their own retention to legal requirements rather than inherit a Mium default.

## The Sweep Loop

The leader Master runs a background loop on a single cadence:

```properties
mium.retention.sweep.interval.seconds = 3600     # default: hourly
```

Each tick, the loop walks the rules below. The loop is leader-only — followers never run it, so a leader change does not double-delete.

Follower Masters and Workers do not need retention configuration; they only need to be able to read the same `mium.properties`.

## Memory (Chat Sessions)

| Key | Default | Meaning |
|---|---|---|
| `mium.memory.ttl.days` | `0` | Days to keep chat sessions. `0` = disabled. |
| `mium.memory.user.maxSessions` | `0` | Per-user session cap. `0` = unlimited. Excess sessions are evicted oldest-first. |
| `mium.memory.session.maxTurns` | `200` | Hard cap on turns inside a single session before `MemoryCompactor` summarises older turns into one synthetic message. |

`maxTurns` is the per-session compactor — it never deletes anything, it just collapses. The TTL and per-user cap actually delete sessions and the messages under them.

Recommended starting point on a regulated on-prem deployment:

```properties
mium.memory.ttl.days          = 50
mium.memory.user.maxSessions  = 100
```

## Prompt Library

| Key | Default | Meaning |
|---|---|---|
| `mium.prompt.ttl.days` | `0` | Days to keep saved prompts. `0` = disabled. |
| `mium.prompt.user.maxPrompts` | `0` | Per-user prompt cap. `0` = unlimited. |
| `mium.prompt.body.maxBytes` | `65536` | Max prompt body size at write time (64 KiB). |

`body.maxBytes` is enforced at insert time — it never deletes existing prompts; it only rejects oversized writes.

## Embeddings

```properties
mium.embedding.ttl.days = 0     # default: disabled
```

When the underlying chat session or prompt is deleted, its embeddings are deleted with it (Mium owns the foreign keys). The independent embedding TTL is a defensive sweep for orphans.

## Exported Files (S3)

```properties
mium.tempfile.ttl.days = 10     # default: 10 days
```

Export ciphertext in S3 is the **only** retention rule that defaults to *enabled*, because exports are intended to be short-lived — the user clicks "Download", the file is read, decrypted, and immediately deleted from S3. The 10-day sweep is a safety net for files that were rendered but never downloaded.

To rely on **S3 lifecycle policies** instead — which is preferable for AWS S3 buckets where the platform owns the retention narrative — set this to `0` and configure a bucket-level expiration rule.

## Bootstrap Tier — Why There Is No TTL

IAM, KMS, and ConnectionStore have no TTL. The data there represents identities and credentials — deleting them is an explicit human action via the Admin UI or `DELETE /api/iam/...` and `DELETE /api/connections`. A retention sweep would be a foot-gun.

## Tuning the Sweep

The defaults are sized for an installation with a few hundred users. For larger deployments:

- Drop `mium.retention.sweep.interval.seconds` to `300` (5 minutes) so sweeps complete before the next tick begins.
- Watch `Retention sweep took N ms` log lines on the leader. A sweep approaching the interval bound means it is overlapping itself — split the workload by raising the TTL or capping per-user counts more aggressively.

## Disabling Retention Entirely

Setting every `*.ttl.days` to `0`, every `*.user.max*` to `0`, and `mium.tempfile.ttl.days = 0` makes the sweep a no-op. This is the right configuration when an external compliance system owns retention and Mium must not delete anything on its own. Operators in this mode usually:

- Keep `mium.memory.session.maxTurns` set so individual sessions stay tractable for the LLM.
- Use S3 bucket lifecycle for export expiration.
- Run their own NeorunBase-side cleanup against the `mium_chat_session` / `mium_prompt` / `mium_embedding` tables on a schedule they control.
