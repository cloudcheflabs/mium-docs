# Storage Backends

Mium splits its persistent state across stores backed by three CCL-stack technologies: embedded RocksDB for bootstrap-critical data, NeorunBase for shared application data, and S3-compatible storage for rendered files.

## Store Matrix

| Store | Backend | Purpose |
|---|---|---|
| IAM | RocksDB | Users, groups, policies, companies, organizations |
| KMS | RocksDB | Versioned envelope encryption keys |
| ConnectionStore | RocksDB | Per-user tool/LLM credentials (KMS-encrypted) |
| MemoryStore | NeorunBase | Per-user chat sessions and messages |
| PromptStore | NeorunBase | Per-user saved prompt library |
| EmbeddingStore | NeorunBase (VECTOR) | Vector embeddings for retrieval |
| TempFileStore | S3-compatible | Encrypted ciphertext of server-rendered exports |

## Why This Split

**IAM / KMS / ConnectionStore → RocksDB**: These stores bootstrap the cluster itself. The leader must seed KMS keys and create the admin user before it can connect to anything external. Embedding them in RocksDB means the cluster starts with zero external dependencies beyond ZooKeeper.

**MemoryStore / PromptStore / EmbeddingStore → NeorunBase**: Chat history, prompts, and embeddings are application-level data shared across all Mium nodes. Storing them in NeorunBase means every Master and Worker reads and writes directly — no Mium-side snapshot replication, no leader bottleneck for writes. NeorunBase handles durability and consistency natively.

**TempFileStore → S3-compatible storage**: Server-rendered export files (XLSX / PDF / PPTX) are produced by Workers and downloaded by users via the Master. S3 is the natural shared medium — the Worker PUTs the encrypted file, the Master GETs and streams it. No file needs to live on a single node's local disk.

## NeorunBase Configuration

```properties
mium.neorunbase.jdbc.url  = jdbc:postgresql://<host>:5432/neorunbase?preferQueryMode=simple
mium.neorunbase.username  = admin
mium.neorunbase.password  = ...
mium.neorunbase.schema    = mium
```

Tables (`mium_chat_session`, `mium_chat_message`, `mium_prompt`, `mium_embedding`) are auto-created on startup. `mium_chat_session` carries a `workspace VARCHAR(32) DEFAULT 'chat'` column so a session belongs to a workspace (`chat` / `analyze` / `dev`). Secondary indexes are deliberately **not** created (NeorunBase secondary-index population is not relied on); the stores use full scans instead.

## S3 Configuration

```properties
mium.tempfile.s3.connection   = s3-default       # ConnectionStore id (tool=s3, authType=access_key)
mium.tempfile.s3.endpoint     = http://localhost:8000
mium.tempfile.s3.region       = us-east-1
mium.tempfile.s3.bucket       = mium-tempfile
mium.tempfile.s3.path.style   = true
```

S3 credentials live in the ConnectionStore (not in `mium.properties`). The bucket is auto-created on first use.

## Replication

Only IAM, KMS, and ConnectionStore are replicated between Mium nodes via internal NIO sync messages. The leader pushes snapshots to followers, and followers self-heal by periodically pulling. Memory, Prompt, Embedding, and TempFile stores do not require Mium-side replication because NeorunBase and S3 handle distribution natively.

## Retention & Data Management

The leader runs a periodic retention sweep (`mium.retention.sweep.interval.seconds`, default hourly) that applies the configured TTLs to the NeorunBase stores (`sweepExpired` on memory, prompt, and embedding), enforces the per-user session cap, and reaps expired S3 temp files. See [Configuration](configuration.md).

Users and admins can also wipe stored data directly via the REST layer:

| Method | Path | Purpose |
|---|---|---|
| `DELETE` | `/api/storage/me` | Wipe the caller's own data. Optional body `{ "kinds": ["memory","prompt","embedding"] }` (default: all three). |
| `DELETE` | `/api/storage/purge` | Admin-only (requires `SYSTEM:MANAGE_IAM` on `system:iam:*`). Wipes the store across all users. |

Per-user embedding deletion is intentionally unsupported (embeddings are keyed on `(namespace, id)` with no owner column) — only the admin purge variant clears embeddings.
