# Connection Management

Mium provides a per-user encrypted credential vault — the **ConnectionStore** — for managing connections to external systems (LLM providers, data tools, object stores). Credentials are encrypted at rest via KMS and are never shared between users.

## ConnectionStore

The ConnectionStore is a RocksDB-backed store on the leader Master that holds per-user connection records:

- Each user manages their own set of connections.
- Auth payloads are encrypted with KMS envelope encryption (AES-256-GCM) — each connection's auth data has its own DEK, the DEK is wrapped under the master KEK.
- The store is replicated from the leader Master to followers via internal NIO sync, so any node can serve reads.

## Connection Types

Mium recognizes a `tool` field on each connection that tells the rest of the system how to interpret the auth payload and where to use the connection.

| Tool | Auth Type | Used By |
|------|-----------|---------|
| `anthropic` | `api-key` | Agent loop — Anthropic Claude chat |
| `ollama` | `api-key` (optional) | Agent loop — local / self-hosted Ollama |
| `ontul` | `access_key` or `token` | Tool layer — Ontul SQL queries and job lifecycle |
| `s3` | `access_key` | Temp File Storage backend (ShannonStore / MinIO / AWS S3) |

Adding a new tool registers a new value here without touching ConnectionStore itself.

## Management

Connections are managed through the Admin UI (Settings → Tool Connections, Settings → LLM Backends) or the REST API at `/admin/api/connections`:

- **Create** — register a new connection with tool type, endpoint, and auth payload.
- **List** — view connections owned by the current user; admins with the right policy can view other users' connections.
- **Delete** — remove a connection.

Updates are leader-only; followers transparently proxy.

## Security

- Auth payloads are encrypted under per-connection DEKs wrapped by the cluster's KMS key — see [Encryption & KMS](encryption.md).
- Connection IDs are referenced everywhere downstream (e.g. `mium.tempfile.s3.connection`) — raw credentials never appear in `mium.properties`, REST responses, or logs.
- Replication of the encrypted store between Masters runs over the internal NIO protocol, which is itself envelope-encrypted once the cluster has a KMS key.
