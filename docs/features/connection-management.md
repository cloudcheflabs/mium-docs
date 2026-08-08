# Connection Management

Mium provides a per-user encrypted credential vault — the **ConnectionStore** — for managing connections to external systems (LLM providers, data tools, object stores). Credentials are encrypted at rest via KMS and are never shared between users.

## ConnectionStore

The ConnectionStore is a RocksDB-backed store on the leader Master that holds per-user connection records:

- Each user manages their own set of connections.
- Auth payloads are encrypted with KMS envelope encryption (AES-256-GCM) — each connection's auth data has its own DEK, the DEK is wrapped under the master KEK.
- The store is replicated from the leader Master to followers via internal NIO sync, so any node can serve reads.

## Connection Types

Mium recognizes a `tool` field on each connection that tells the rest of the system how to interpret the auth payload and where to use the connection.

| Tool | Managed in | Used By |
|------|-----------|---------|
| `anthropic` | Settings → LLM Backends | Agent loop — Anthropic Claude chat |
| `ollama` | Settings → LLM Backends | Agent loop — local / self-hosted Ollama |
| `ontul` | Settings → Tool Connections | Tool layer — Ontul SQL queries and job lifecycle |
| `s3` | Settings → Tool Connections | Temp File Storage backend (ShannonStore / MinIO / AWS S3) |

Adding a new tool registers a new value here without touching ConnectionStore itself.

Each connection also carries an `authType` string describing its auth payload. The canonical constants defined on `Connection` are `token`, `access_key`, `basic`, and `oauth2`; readers such as the Ontul client also infer the scheme directly from the payload keys, so the value is descriptive rather than strictly enforced. In the shipped UI, LLM-backend connections are saved with `authType=api-key` and Tool Connections (Ontul / S3) with `authType=static`. Ontul / S3 payloads use `accessKeyId` + `secretAccessKey` (the legacy `secretKey` spelling is also accepted).

## Management

Connections are managed through the Admin UI (Settings → Tool Connections, Settings → LLM Backends) or the REST API at `/api/connections` (the `/admin` prefix applies only when `mium.admin.context.path` is set):

- **Create** — register a new connection with tool type, endpoint, and auth payload.
- **List** — view connections owned by the current user; admins with the right policy can view other users' connections. Auth payloads are never returned.
- **Delete** — remove a connection; the `connectionId` is passed in the request body (`DELETE /api/connections` with `{connectionId}`), not as a path segment.

Updates are leader-only; followers transparently proxy.

## Security

- Auth payloads are encrypted under per-connection DEKs wrapped by the cluster's KMS key (`mium-connection`); the outer RocksDB snapshot and NIO sync blob are encrypted under a separate `mium-connection-snapshot` key, so the two layers rotate independently — see [Encryption & KMS](encryption.md).
- Connection IDs are referenced everywhere downstream (e.g. `mium.tempfile.s3.connection`) — raw credentials never appear in `mium.properties`, REST responses, or logs.
- Replication of the encrypted store between Masters runs over the internal NIO protocol, which is itself envelope-encrypted once the cluster has a KMS key.
