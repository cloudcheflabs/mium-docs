# Encryption & KMS

Mium provides built-in encryption with a Key Management Service (KMS) to protect sensitive data at rest and in transit between cluster nodes.

## Envelope Encryption

Each piece of sensitive data is encrypted with its own Data Encryption Key (DEK). DEKs are wrapped under the master Key Encryption Key (KEK):

- **Algorithm** — AES-256-GCM.
- **Master key derivation** — PBKDF2-SHA256 with 200,000 iterations.
- **Master key source** — environment variable named by `mium.kms.master.key.env` (default `MIUM_MASTER_KEY`); the value must be at least 32 characters.
- **Versioned KEKs** — Key Encryption Keys are versioned. Rotation produces a new active version while old versions stay available for decrypting historical data.

## What Is Encrypted

| Surface | Mechanism |
|---|---|
| ConnectionStore auth payloads | Per-connection envelope (`mium-connection` key); the outer RocksDB snapshot / NIO sync blob uses a separate `mium-connection-snapshot` key |
| IAM snapshots on disk | Envelope (`mium-iam` key) |
| Server-rendered export ciphertext (XLSX / PDF / PPTX) | Envelope (`mium-tempfile` key) |
| Internal NIO control-plane payloads | Once a KMS key is available, internal messages between Master ↔ Master and Master ↔ Worker are envelope-wrapped before transmission |

Mium is explicit about TLS-handled boundaries — the browser ↔ master HTTP path is expected to terminate TLS at the deployment's reverse proxy / nginx, and is not double-wrapped.

Chat history, prompts, and embeddings live in **NeorunBase** and are **not** KMS-envelope-encrypted by Mium (prompts are explicitly not treated as credentials); protect those at the NeorunBase / transport layer.

## Built-in KMS

`MiumKmsProvider` is the in-process KMS:

- Stores the versioned KEK bundle in a RocksDB-backed keystore (`mium.kms.rocksdb.path`).
- Supports versioned keys and on-demand rotation.
- Replicates the keystore from the leader to follower Masters **and Workers** via the internal NIO `KMS_SYNC` opcode. Each node imports the entire keystore bundle (all KEK versions for all key ids) in one hop — Workers receive the full keystore, not a per-key subset.

No external KMS service is required. Operators who want to integrate a corporate KMS can implement the same provider interface.

## Key Distribution

1. The leader Master generates versioned KEKs and stores them in the local encrypted RocksDB keystore. Per-encryption DEKs are generated on the fly and wrapped by the active KEK — they are never persisted separately.
2. On leader election or key changes, the whole keystore is replicated to followers and Workers via `KMS_SYNC`.
3. Workers hold the full keystore, so they can unwrap the DEKs protecting connection credentials at tool-execution time.

## Configuration

```properties
mium.kms.enabled            = true
mium.kms.master.key.env     = MIUM_MASTER_KEY
mium.kms.rocksdb.path       = ${mium.base.data.dir}/kms
mium.kms.pbkdf2.iterations  = 200000
```

## Management

KMS keys are managed through the Admin UI (Settings → Administration → Security & KMS) or the REST API under `/api/kms/*` — `GET /api/kms/list`, `GET /api/kms/status/{keyId}`, `POST /api/kms/create`, and `POST /api/kms/rotate/{keyId}` (the `/admin` prefix applies only when `mium.admin.context.path` is set).
