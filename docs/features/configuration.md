# Configuration

Mium is configured through a single `mium.properties` file, with overrides available from the environment or the JVM command line.

## Precedence

For any key, the highest-priority source wins:

```
System Property (-D) > Environment Variable > properties file
```

So a value baked into `mium.properties` can be overridden by an environment variable, which in turn can be overridden by a `-D` system property passed to the JVM.

## Environment Variable Naming

To set a property via an environment variable, take the property key, replace every `.` with `_`, and uppercase it:

| Property key | Environment variable |
|---|---|
| `mium.master.admin.port` | `MIUM_MASTER_ADMIN_PORT` |
| `mium.zk.connect` | `MIUM_ZK_CONNECT` |
| `mium.kms.master.key.env` | `MIUM_KMS_MASTER_KEY_ENV` |

## Base

| Property | Default | Description |
|---|---|---|
| `mium.base.data.dir` | `./data` | Root directory for all node-local persistent state (KMS, IAM, ConnectionStore RocksDB stores, admin recovery socket, IAM audit log). Referenced by `${mium.base.data.dir}` in many keys below. In containers this must be an absolute, mounted volume path or state is lost on restart. |

## ZooKeeper

ZooKeeper is the coordination backbone — leader election (Curator `LeaderLatch`) and the master/worker `NodeRegistry` both live here.

| Property | Default | Description |
|---|---|---|
| `mium.zk.connect` | `localhost:2181` | Comma-separated `host:port` connect string for the ZooKeeper ensemble. |
| `mium.zk.root` | `/mium` | Base znode namespace for all Mium coordination state (e.g. `/mium/leader`, `/mium/nodes`). Give each cluster sharing one ensemble a distinct root. |
| `mium.zk.session.timeout.ms` | `30000` | ZK session timeout in milliseconds. If a node fails to heartbeat within this window the session expires and it loses leadership / registry entries. Higher tolerates GC pauses but slows failover detection. |
| `mium.zk.connect.timeout.ms` | `10000` | Max milliseconds to wait when first establishing the ZK connection at startup. Should stay well below the session timeout. |

## Master

| Property | Default | Description |
|---|---|---|
| `mium.master.host` | `0.0.0.0` | Bind address for both master servers (admin HTTP and internal NIO control plane). `0.0.0.0` binds all interfaces; pin to a NIC to restrict exposure. |
| `mium.master.admin.port` | `8090` | TCP port for the admin HTTP server — serves the admin UI, REST API (`/api/*`), and `/health` + `/ready` probes. The port users and the UI talk to; also part of this master's node id. |
| `mium.master.internal.port` | `19099` | TCP port for the master's internal NIO control plane. Carries master-to-master peer RPC (metrics polling, KMS key-seed broadcast, follower→leader coordination). Keep on a trusted network. |
| `mium.admin.context.path` | _(empty)_ | URL prefix the entire HTTP surface is mounted under. Default `""` = root. When changed (e.g. `/admin`), the UI bundle must be rebuilt with `VITE_BASE_PATH` set to the same prefix. |

## Worker

| Property | Default | Description |
|---|---|---|
| `mium.worker.host` | `0.0.0.0` | Bind address the worker advertises and listens on for its internal NIO control plane. `0.0.0.0` binds all interfaces. |
| `mium.worker.internal.port` | `19098` | TCP port for the worker's internal NIO server. The master dispatches work (Python-subprocess export jobs, embedding daemons) over this port. Also forms the worker's node id. Not HTTP, not user-facing — keep on a trusted network. |

## KMS

The envelope-encryption Key Management Service. See [Encryption & KMS](encryption.md).

| Property | Default | Description |
|---|---|---|
| `mium.kms.enabled` | `true` | Master switch for the envelope-encryption KMS. When true, a RocksDB-backed KMS with versioned KEKs initialises before any at-rest store opens. Setting false disables encryption-at-rest entirely — only safe for throwaway dev. |
| `mium.kms.rocksdb.path` | `${mium.base.data.dir}/kms` | Filesystem directory for the KMS RocksDB instance (wrapped KEKs and key metadata). |
| `mium.kms.master.key.env` | `MIUM_MASTER_KEY` | Name of the **environment variable** that holds the actual master key (the env-var name, not the value). The KMS reads it at startup and derives the root key via PBKDF2. Must be set on every node or the master fails fast. |
| `mium.kms.pbkdf2.iterations` | `200000` | PBKDF2 iteration count used to stretch the master-key value into the root key. Must be identical across restarts and all nodes; changing it invalidates existing KMS material. |

## IAM

See [IAM](iam.md).

| Property | Default | Description |
|---|---|---|
| `mium.iam.rocksdb.path` | `${mium.base.data.dir}/iam` | Filesystem directory for the IAM RocksDB instance — users, password hashes, policies, and group/organization membership. |
| `mium.iam.admin.user` | `admin` | Username of the bootstrap admin account. Created only on first boot; changing it later does not rename the existing account. |
| `mium.iam.admin.password` | `admin` | Initial password for the bootstrap admin account, applied only on first-boot seeding. **Change this before any real deployment**, or rotate it afterwards. Editing it after first boot has no effect. |

## ConnectionStore

| Property | Default | Description |
|---|---|---|
| `mium.connection.rocksdb.path` | `${mium.base.data.dir}/connections` | Filesystem directory for the ConnectionStore RocksDB instance. Holds per-user external-tool credentials (Ontul, Snowflake, S3, LLM API keys). Auth payloads are KMS-encrypted at rest. |

## NeorunBase (required)

NeorunBase is a hard dependency — the per-user chat Memory store, Prompt store, and Embedding (VECTOR) store all live in it. The master fails fast at startup if it cannot reach NeorunBase.

| Property | Default | Description |
|---|---|---|
| `mium.neorunbase.jdbc.url` | `jdbc:postgresql://localhost:5432/neorunbase?preferQueryMode=simple` | Postgres-wire JDBC URL of the NeorunBase instance. `preferQueryMode=simple` is required for wire compatibility — keep it on. |
| `mium.neorunbase.username` | `admin` | JDBC username used to connect to NeorunBase. |
| `mium.neorunbase.password` | _(empty)_ | JDBC password for the NeorunBase user. Blank = no password. |
| `mium.neorunbase.schema` | `mium` | Schema/namespace inside NeorunBase under which Mium's tables (memory, prompts, embeddings) are created. Isolates Mium's data from other tenants. |

## Embedding

| Property | Default | Description |
|---|---|---|
| `mium.embedding.dim` | `768` | Vector dimension for the NeorunBase embedding store; becomes the width of the `VECTOR(dim)` column. Must exactly match the embedding model's output dimension. Cannot be changed after the table is created without recreating it. The embedding store is only wired up when `mium.embedding.enabled` is true (defaults to false). |

## Retention

All TTLs default to `0` = retention sweep **disabled**; operators set days explicitly to opt in. Sweep cadence is governed by `mium.retention.sweep.interval.seconds`. Recommended on-prem values: memory 50 days, prompt 100 days, embedding 50 days.

| Property | Default | Description |
|---|---|---|
| `mium.memory.ttl.days` | `0` | Chat-memory retention in days. `0` = sweep disabled. When > 0, the leader deletes sessions whose `updatedAt` is older than this. |
| `mium.prompt.ttl.days` | `0` | Prompt-store retention in days. `0` = sweep disabled. When > 0, prompts older than this are deleted. |
| `mium.embedding.ttl.days` | `0` | Embedding-store retention in days. `0` = sweep disabled. Global age sweep keyed on `(namespace, id)` — no per-user path. |
| `mium.memory.user.maxSessions` | `0` | Per-user cap on retained chat sessions. `0` = unlimited. When > 0, the oldest sessions beyond the cap are pruned. |
| `mium.prompt.user.maxPrompts` | `0` | Per-user cap on stored prompts. `0` = unlimited. When > 0, `createPrompt` is rejected once the user is at the cap. |
| `mium.memory.session.maxTurns` | `200` | Max message turns kept per chat session (a count, not time). When exceeded, the oldest turns are dropped. Bounds per-session memory growth and LLM context size. |
| `mium.prompt.body.maxBytes` | `65536` | Max size in bytes of a single prompt's body (64 KiB). create/update is rejected if exceeded. |
| `mium.retention.sweep.interval.seconds` | `3600` | How often the leader runs the retention sweep, in seconds (hourly). Governs every TTL sweep above plus the S3 temp-file sweep. Only the leader runs it. |

## JWT

| Property | Default | Description |
|---|---|---|
| `mium.jwt.ttl.seconds` | `86400` | Lifetime of issued auth JWTs in seconds (24 hours). After this, tokens expire and clients must re-authenticate. |

## S3 Temp File Storage (required)

Server-rendered export ciphertext (XLSX / PDF / PPTX) is stored in S3. Credentials live in the ConnectionStore (`tool=s3`, `authType=access_key`). See [Temp File Storage](temp-file-storage.md).

| Property | Default | Description |
|---|---|---|
| `mium.tempfile.filename.prefix` | `mium-export` | Filename prefix applied to every generated export object key in S3. Cosmetic/organisational — makes Mium's objects easy to spot and target with lifecycle rules. |
| `mium.tempfile.kms.key.id` | `mium-tempfile` | KMS key id used to envelope-encrypt export ciphertext before upload to S3. Resolved against the configured KMS. |
| `mium.tempfile.s3.connection` | `s3-default` | ConnectionStore connection id (`tool=s3`, `authType=access_key`) supplying the S3 access key / secret. Only the id is referenced here; the credentials live in the ConnectionStore. |
| `mium.tempfile.s3.endpoint` | `http://localhost:8000` | S3 endpoint URL for the temp-file bucket. Point at AWS S3, MinIO, or ShannonStore. |
| `mium.tempfile.s3.region` | `us-east-1` | S3 region for the temp-file bucket. For non-AWS stores, any non-empty value is typically accepted. |
| `mium.tempfile.s3.bucket` | `mium-tempfile` | Name of the S3 bucket holding exported temp files. Must exist or be creatable with the supplied credentials. |
| `mium.tempfile.s3.path.style` | `true` | Use path-style S3 addressing instead of virtual-host style. `true` is required for MinIO/ShannonStore and most non-AWS endpoints; AWS S3 supports either. |
| `mium.tempfile.ttl.days` | `10` | Days to keep exported temp files in S3. Files older than this are swept by the housekeeping loop. `0` = sweep disabled (use S3 lifecycle rules instead). Default keeps chat-shared download links valid for ~a week. |

## LLM Backend HTTP

Shared HTTP knobs for every LLM/embedding backend wired through `LlmBackendFactory` (Anthropic, Ollama, Ollama-embed). Per-call `LlmRequest.timeoutMs` still wins when set — these are the factory defaults.

| Property | Default | Description |
|---|---|---|
| `mium.llm.http.connect.timeout.ms` | `10000` | Connect timeout in milliseconds for LLM backend HTTP calls. |
| `mium.llm.http.request.timeout.ms` | `60000` | Request timeout in milliseconds for LLM backend HTTP calls. |
| `mium.llm.embedding.request.timeout.ms` | `60000` | Per-request timeout in milliseconds for embedding calls, which are batchy and not driven by `LlmRequest`. |

## Admin HTTP Follower → Leader Proxy

When a master is a follower, write routes are reverse-proxied to the current leader. See [High Availability](high-availability.md).

| Property | Default | Description |
|---|---|---|
| `mium.admin.proxy.connect.timeout.ms` | `2000` | Connect timeout in milliseconds for the follower→leader proxy. Short by design — master nodes share a LAN. |
| `mium.admin.proxy.request.timeout.ms` | `10000` | Request timeout in milliseconds for the follower→leader proxy. Covers the slowest synchronous IAM / Connection admin call. |

## Optional Properties

These keys are not listed in the shipped `mium.properties` but are honoured when set:

| Property | Default | Description |
|---|---|---|
| `mium.embedding.enabled` | `false` | When true, wires up the NeorunBase embedding (VECTOR) store. Required before `mium.embedding.dim` and embedding TTL take effect. |
