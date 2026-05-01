# Configuration Reference

All Mium settings live in `conf/mium.properties`. Keys can be overridden in three ways, in precedence order:

1. **System property** — `-Dmium.master.admin.port=9090` on the JVM command line.
2. **Environment variable** — replace `.` with `_` and uppercase: `mium.master.admin.port` → `MIUM_MASTER_ADMIN_PORT`.
3. **`conf/mium.properties`** — the file shipped with the distribution.

Property values can reference other keys with `${...}`:

```properties
mium.kms.rocksdb.path = ${mium.base.data.dir}/kms
```

## Base

| Key | Default | Description |
|---|---|---|
| `mium.base.data.dir` | `./data` | Root directory for the embedded RocksDB stores (KMS, IAM, ConnectionStore). All other RocksDB paths are derived from this. |
| `mium.home` | (set by launcher) | Package install root. The launcher scripts inject this automatically. |

## ZooKeeper

| Key | Default | Description |
|---|---|---|
| `mium.zk.connect` | `localhost:2181` | ZooKeeper connect string. Comma-separated list for an ensemble. |
| `mium.zk.root` | `/mium` | Root ZK path under which Mium creates its `masters/`, `workers/`, and readiness znodes. |
| `mium.zk.session.timeout.ms` | `30000` | ZK session timeout. Determines failover bound. |
| `mium.zk.connect.timeout.ms` | `10000` | Initial ZK connect timeout. |

## Master

| Key | Default | Description |
|---|---|---|
| `mium.master.host` | `0.0.0.0` | Bind address for the Master HTTP and NIO listeners. |
| `mium.master.admin.port` | `8090` | Admin HTTP port (Admin UI + REST API). |
| `mium.master.internal.port` | `19099` | Master internal NIO port (Master ↔ Master sync, Worker → Master heartbeat). |
| `mium.admin.context.path` | (empty) | URL prefix the entire HTTP surface is mounted under. Default empty = root: UI at `http://host:port/`, REST at `/api/*`. Set to e.g. `/admin` or `/mium` when fronting Mium with a path-based reverse proxy. The Admin UI bundle must be rebuilt with a matching `VITE_BASE_PATH`. |
| `mium.admin.ui.dir` | `${mium.home}/admin-ui` | Override the path the Master serves the Admin UI bundle from. |

## Worker

| Key | Default | Description |
|---|---|---|
| `mium.worker.host` | `0.0.0.0` | Bind address for the Worker NIO listener. |
| `mium.worker.internal.port` | `19098` | Worker NIO port (receives `EXECUTE_AGENT` / `EXECUTE_TOOL` / `EXECUTE_EXPORT` from Masters). |

## KMS

| Key | Default | Description |
|---|---|---|
| `mium.kms.enabled` | `true` | Toggle envelope encryption. Turning this off is **not** supported in production — most stores assume encryption is available. |
| `mium.kms.rocksdb.path` | `${mium.base.data.dir}/kms` | RocksDB path for the encrypted KEK bundle. |
| `mium.kms.master.key.env` | `MIUM_MASTER_KEY` | Name of the environment variable that holds the 32+ char master secret. |
| `mium.kms.pbkdf2.iterations` | `200000` | PBKDF2-SHA256 iteration count for deriving the root KEK from the master secret. |

## IAM

| Key | Default | Description |
|---|---|---|
| `mium.iam.rocksdb.path` | `${mium.base.data.dir}/iam` | RocksDB path for users, groups, policies, companies, organizations. |
| `mium.iam.admin.user` | `admin` | Bootstrap admin username. Created only if the IAM store is empty. |
| `mium.iam.admin.password` | `admin` | Bootstrap admin password. **Rotate immediately after first login.** |
| `mium.jwt.ttl.seconds` | `86400` | JWT bearer token lifetime. The browser auto-refreshes via `/auth/refresh` before expiry. |

## ConnectionStore

| Key | Default | Description |
|---|---|---|
| `mium.connection.rocksdb.path` | `${mium.base.data.dir}/connections` | RocksDB path for per-user encrypted connection records (LLM keys, Ontul access keys, S3 access keys). |

## NeorunBase (Memory / Prompt / Embedding)

| Key | Default | Description |
|---|---|---|
| `mium.neorunbase.jdbc.url` | `jdbc:postgresql://localhost:5432/neorunbase?preferQueryMode=simple` | NeorunBase JDBC URL. **Required** — the platform refuses to start if this is unreachable. |
| `mium.neorunbase.username` | `admin` | NeorunBase username. |
| `mium.neorunbase.password` | (empty) | NeorunBase password. |
| `mium.neorunbase.schema` | `mium` | Schema in which Mium auto-creates `mium_chat_session`, `mium_chat_message`, `mium_prompt`, `mium_embedding`. |
| `mium.embedding.dim` | `768` | Vector column dimension for the EmbeddingStore. Must match the embedding model in use. |

## Retention

All TTLs default to `0` = sweep disabled. Operators opt in with explicit non-zero values. The sweep cadence is `mium.retention.sweep.interval.seconds`.

| Key | Default | Description |
|---|---|---|
| `mium.memory.ttl.days` | `0` | Days to keep chat sessions. `0` disables the sweep. |
| `mium.prompt.ttl.days` | `0` | Days to keep saved prompts. |
| `mium.embedding.ttl.days` | `0` | Days to keep embeddings. |
| `mium.memory.user.maxSessions` | `0` | Max chat sessions per user. `0` = unlimited. Oldest sessions are evicted past this cap. |
| `mium.prompt.user.maxPrompts` | `0` | Max saved prompts per user. `0` = unlimited. |
| `mium.memory.session.maxTurns` | `200` | Hard cap on turns per chat session before `MemoryCompactor` summarises older turns. |
| `mium.prompt.body.maxBytes` | `65536` | Max prompt body size at write time (64 KiB). |
| `mium.retention.sweep.interval.seconds` | `3600` | How often the leader runs the retention sweep. |

## Server-Side Export (S3 Temp Files)

The export pipeline (XLSX / PDF / PPTX) runs on Workers, ciphertext is uploaded to an S3-compatible bucket, and the Master streams it back to the user at download time. Credentials live in the ConnectionStore — `mium.properties` only carries the bucket coordinates.

| Key | Default | Description |
|---|---|---|
| `mium.tempfile.filename.prefix` | `mium-export` | Object key prefix in the bucket. |
| `mium.tempfile.kms.key.id` | `mium-tempfile` | KMS key id used for envelope encryption. |
| `mium.tempfile.s3.connection` | `s3-default` | ConnectionStore id (`tool=s3`, `authType=access_key`) holding bucket access key + secret. |
| `mium.tempfile.s3.endpoint` | `http://localhost:8000` | S3 endpoint. ShannonStore's default is `:8000`; AWS S3 is `https://s3.<region>.amazonaws.com`. |
| `mium.tempfile.s3.region` | `us-east-1` | S3 region. |
| `mium.tempfile.s3.bucket` | `mium-tempfile` | Bucket name. Auto-created on first use if the credentials allow it. |
| `mium.tempfile.s3.path.style` | `true` | Use path-style addressing (`endpoint/bucket/key`) instead of virtual-hosted (`bucket.endpoint/key`). MinIO and ShannonStore require path-style. |
| `mium.tempfile.ttl.days` | `10` | Days a rendered export survives before the leader sweep deletes it. `0` disables the sweep — use S3 lifecycle rules instead. |

## Catalog Filtering (Ontul)

When the agent introspects an Ontul connection's catalog, these properties hide system / sample catalogs and schemas from the LLM context.

| Key | Default | Description |
|---|---|---|
| `mium.catalog.deny.catalogs` | `tpch,tpcds,system,information_schema` | Comma-separated catalog names hidden from the agent. |
| `mium.catalog.deny.schemas` | `information_schema` | Comma-separated schema names hidden from the agent. |

## Logging

| Key | Default | Description |
|---|---|---|
| `mium.log.path` | `logs` | Log directory. Relative paths resolve under `MIUM_HOME`. |
| `mium.log.output.name` | `<service>-<port>.out` (set by launcher) | File the launcher redirects stdout to. Logback writes to stdout only — the `.out` file is the authoritative log on disk. |

## JVM Options (`conf/jvm.conf`)

`conf/jvm.conf` is a flat list of JVM options, one per line. Lines starting with `#` are comments. The launcher reads the file at start time and prepends every entry to the JVM command line. The same file applies to both Master and Worker — for per-role overrides set `JAVA_OPTS` in the environment.

A safe baseline:

```
-Xms4g
-Xmx4g
-XX:MaxDirectMemorySize=4g
-XX:+UseG1GC
-XX:+UseStringDeduplication
-XX:+OptimizeStringConcat
--add-opens=java.base/java.nio=ALL-UNNAMED
-Dio.netty.noPreferDirect=true
```

The `--add-opens` line is required on Java 17+ for Apache Arrow to access internal `java.nio` modules.
