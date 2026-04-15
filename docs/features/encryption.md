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
| ConnectionStore auth payloads | Per-connection envelope (`mium-connection` key) |
| IAM snapshots on disk | Envelope (`mium-iam` key) |
| MemoryStore RocksDB snapshot (when `backend=rocksdb`) | Envelope (`mium-memory` key) |
| PromptStore RocksDB snapshot (when `backend=rocksdb`) | Envelope (`mium-prompt` key) |
| Server-rendered export ciphertext (XLSX / PDF / PPTX) | Envelope (`mium-tempfile` key) |
| Internal NIO control-plane payloads | Once a KMS key is available, internal messages between Master ↔ Master and Master ↔ Worker are envelope-wrapped before transmission |

Mium is explicit about TLS-handled boundaries — the browser ↔ master HTTP path is expected to terminate TLS at the deployment's reverse proxy / nginx, and is not double-wrapped.

## Built-in KMS

`MiumKmsProvider` is the in-process KMS:

- Stores the encrypted KEK bundle in a RocksDB-backed keystore (`mium.kms.rocksdb.path`).
- Supports versioned keys and on-demand rotation.
- Replicates the keystore from the leader to follower Masters via the internal NIO `KMS_SYNC` opcode. Workers receive only the keys they need to decrypt the connection payloads they are dispatched to use.

No external KMS service is required. Operators who want to integrate a corporate KMS can implement the same provider interface.

## Key Distribution

1. The leader Master generates and stores DEKs in the local encrypted RocksDB keystore.
2. On leader election or key changes, the keystore is replicated to followers via `KMS_SYNC`.
3. Workers receive the relevant DEKs needed to decrypt connection credentials at tool-execution time.

## Configuration

```properties
mium.kms.enabled            = true
mium.kms.master.key.env     = MIUM_MASTER_KEY
mium.kms.rocksdb.path       = ${mium.base.data.dir}/kms
mium.kms.pbkdf2.iterations  = 200000
```

## Management

KMS keys are managed through the Admin UI (Settings → Administration → Security & KMS) or the REST API under `/admin/api/kms/*` — list, rotate, and inspect key versions.
