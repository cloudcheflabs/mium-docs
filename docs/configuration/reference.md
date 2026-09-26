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
| `mium.embedding.dim` | `1024` | Vector column dimension for the EmbeddingStore. Must match the embedding model in use — bge-m3 is 1024, all-MiniLM-L6-v2 is 384, CLIP is 512. Changing it requires `mium.mium_embedding` to be dropped and rebuilt, because the column is typed `VECTOR(N)`. |

## Embedding (Semantic Recall)

Off by default: the Python daemons hold several GB of RAM, and most deployments have not sized worker nodes for that. The SPI is a no-op until it is turned on. The model can also be changed at runtime from **Settings → Embedding**, which overrides these bootstrap values.

| Key | Default | Description |
|---|---|---|
| `mium.embedding.enabled` | `false` | Wire `NeorunBaseEmbeddingStore` + `WorkerEmbeddingBackend` into `EmbeddingResources`. |
| `mium.embedding.clip.enabled` | `false` | Also wire a CLIP backend for image embedding. Independent of the text backend, so a text-only deployment does not pay CLIP's memory cost. Requires `mium.embedding.enabled=true`. |
| `mium.embedding.python` | `python3` | Interpreter the Worker execs for the daemons. Point at a venv (`/opt/mium/embed-venv/bin/python`) when the system Python is not the one carrying the libraries. |
| `mium.embedding.hf.home` | (empty) | `HF_HOME` for the daemons — the HuggingFace cache directory. Point it at a mounted volume so a container restart does not re-download the weights (bge-m3 is ~2.2 GB, CLIP ~600 MB). Empty = the daemon's default `~/.cache/huggingface`. |

## Agent Loop

A turn is a small investigation, not a single translation: asked whether a Kafka connector is registered, the honest sequence is to list the connections, look, and only then answer. These two knobs bound how far the agent may go before handing control back.

| Key | Default | Description |
|---|---|---|
| `mium.agent.max.iterations` | `8` | Maximum steps in one turn. Each step is one LLM call plus one tool dispatch, and each sees the previous result. Four is usually enough (look up, narrow, act, report); the rest is room for the turns that check their work. |
| `mium.agent.loop.timeout.ms` | `120000` | Wall-clock ceiling for the whole turn across all steps. A runaway loop is worse than an incomplete answer, but the bound has to leave room for the steps above. |

## Develop Schema Context

How much raw schema the Develop code router is shown. Analyze is answered from the semantic layer alone; Develop cannot be, because the table a job is built against is usually the new one with no view over it yet. See [Job Lifecycle](../features/job-lifecycle.md).

| Key | Default | Description |
|---|---|---|
| `mium.dev.schema.tables.max` | `120` | Table names listed. One round trip covers all of them, so this is a context budget rather than a latency one. |
| `mium.dev.schema.describe.max` | `6` | Tables whose columns are filled in — only those the prompt actually names, because each is its own `DESCRIBE` round trip against the cluster. |

## LLM / Embedding HTTP

Defaults `LlmBackendFactory` hands every backend (Anthropic, Ollama, Ollama-embed, …). A per-call `LlmRequest.timeoutMs` still wins when set.

| Key | Default | Description |
|---|---|---|
| `mium.llm.http.connect.timeout.ms` | `10000` | TCP connect timeout to the LLM endpoint. |
| `mium.llm.http.request.timeout.ms` | `60000` | Per-request timeout for a chat completion. |
| `mium.llm.embedding.request.timeout.ms` | `60000` | Separate timeout for embedding calls, which are batchy and not driven by `LlmRequest`. |

## Master-to-Master Admin Proxy

A non-leader master forwards write-side admin calls to the current leader. The connect timeout is short by design — master nodes share a LAN; the request timeout covers the slowest synchronous IAM / Connection admin call.

| Key | Default | Description |
|---|---|---|
| `mium.admin.proxy.connect.timeout.ms` | `2000` | Connect timeout to the leader. |
| `mium.admin.proxy.request.timeout.ms` | `10000` | Request timeout for the proxied call. |

## Local Admin Socket (Password Recovery)

A Unix domain socket the master binds for local administrative recovery — see [Admin Password Recovery](../features/admin-password-recovery.md). The socket file's mode-600 permission is the **only** authentication: any process able to connect already shares the master process's filesystem identity.

| Key | Default | Description |
|---|---|---|
| `mium.admin.socket.enabled` | `true` | Bind the socket. Set `false` to remove the local recovery path entirely. |
| `mium.admin.socket.path` | `${mium.base.data.dir}/admin.sock` | Where the socket is bound. Must be on a local filesystem that supports Unix domain sockets — not NFS. Recreated on every master start. |
| `mium.admin.socket.marker.file` | `master.socket` | File, under the data dir, into which the master records the path it actually bound. Re-deriving that path from this file is unreliable: `mium.base.data.dir` can be overridden with `-D` at launch, and nothing else records which value the live process used. Removed on shutdown. |

## Governance Surfaces

| Key | Default | Description |
|---|---|---|
| `mium.instructions.max.chars` | `4000` | Cap on the workspace instruction text injected into every prompt. Text past the limit is dropped with a line saying so, rather than ending mid-rule — a half-written rule is worse than a missing one, because the model still tries to follow it. |
| `mium.verified.list.max` | `200` | Verified pairs returned in one listing. This is the analyst's review queue, not an export. |
| `mium.benchmark.history.max` | `50` | Past benchmark runs shown in the history. Runs are retained regardless; this only bounds what the screen asks for. |
| `mium.benchmark.numeric.tolerance` | `1e-9` | Relative tolerance when comparing a benchmark's numeric answer to its expected value. Two engines summing the same column in a different grouping order disagree in the last bits, and failing on that would make the suite noise. Much larger starts hiding real disagreements — a wrong filter moves a total by orders of magnitude more than this. |

## Authentication — Password Storage

| Key | Default | Description |
|---|---|---|
| `mium.auth.password.hash.iterations` | `200000` | PBKDF2-HMAC-SHA256 iterations used when a local password is written. Mium has always hashed passwords with a per-user salt, so this is a cost knob rather than a migration. The count travels with each stored hash, so raising it does **not** invalidate existing passwords. |

## Single Sign-On (OIDC / SAML / LDAP)

Every setting below can also be managed from the console under **Settings → Single
Sign-On**, which stores it in the `mium_settings` row and applies it on every node
without a restart. **Stored settings win over the properties file**: the file brings
a cluster up, and the console is how it is changed afterwards — if the file won, a
console change would be reverted by the next restart, silently. See
[Single Sign-On](../features/sso.md).

### Identity mapping (all three providers)

| Key | Default | Description |
|---|---|---|
| `mium.sso.group.mappings` | (empty) | `idpGroup:miumGroup` pairs, comma-separated. Empty means provider group names are used as they are. **Once set the mapping is exhaustive** — a group not named here is dropped, so creating a group at the provider cannot grant access on this cluster by itself. |
| `mium.sso.allow.unmapped.groups` | `false` | Whether an identity whose groups all map to nothing may still sign in. Off deliberately: such a session has no policies and is denied every action, so admitting it produces someone signed in who can do nothing. The directory endpoint reports that case as `403`, separately from a wrong password's `401`. |
| `mium.sso.federated.session.seconds` | `3600` | Lifetime of a federated session — the record that lets masters and workers resolve a federated caller's groups by name. It bounds how long access outlives a revocation at the provider, which this cluster is not told about. A federated session cannot be refreshed, for the same reason. |

### OpenID Connect

| Key | Default | Description |
|---|---|---|
| `mium.sso.oidc.enabled` | `false` | Enable the OIDC provider. |
| `mium.sso.oidc.issuer` | (empty) | Issuer URL. Endpoints and the signing key set are read from its discovery document, so they are not configured individually. |
| `mium.sso.oidc.client.id` | (empty) | Client id registered at the provider. |
| `mium.sso.oidc.client.secret` | (empty) | Client secret. Credential — never read back by the console. |
| `mium.sso.oidc.redirect.uri` | `http://localhost:8080/auth/sso/oidc/callback` | Must match the redirect URI registered at the provider exactly, and must be the address browsers reach — the load balancer's, not one master's. |
| `mium.sso.oidc.scopes` | `openid profile email` | Scopes requested. Deliberately excludes `groups`: it is not a standard scope, and a provider that does not define it rejects the whole authorization request with `invalid_scope`. |
| `mium.sso.oidc.username.claim` | `preferred_username` | Claim holding the login name. |
| `mium.sso.oidc.groups.claim` | `groups` | Claim holding group membership. The provider must be configured to include it. |
| `mium.sso.oidc.audience` | (empty) | Expected audience. Empty falls back to the client id. A token issued for another application is refused even though it is genuine and correctly signed. |

### SAML 2.0

| Key | Default | Description |
|---|---|---|
| `mium.sso.saml.enabled` | `false` | Enable the SAML provider. |
| `mium.sso.saml.idp.entity.id` | (empty) | Identity provider entity ID. Read automatically when the provider's metadata is imported from the console. |
| `mium.sso.saml.idp.sso.url` | (empty) | IdP single sign-on URL. |
| `mium.sso.saml.idp.certificate` | (empty) | Base64 IdP signing certificate. Every assertion's signature is verified against it. |
| `mium.sso.saml.sp.entity.id` | `mium` | This cluster's entity ID, as it appears in the SP metadata the provider imports. |
| `mium.sso.saml.sp.acs.url` | `http://localhost:8080/auth/sso/saml/acs` | Assertion consumer URL. As with the OIDC redirect, this must be the address browsers reach. |
| `mium.sso.saml.nameid.format` | (empty) | Requested NameID format. Empty omits the request entirely and lets the provider issue what it is configured for — naming one breaks more integrations than it fixes. |
| `mium.sso.saml.sign.requests` | `false` | Sign authentication requests. Needs an SP keypair, generated from the console; re-import the SP metadata at the provider afterwards so it picks up the certificate. |
| `mium.sso.saml.username.attribute` | `uid` | Assertion attribute holding the login name. |
| `mium.sso.saml.groups.attribute` | `groups` | Assertion attribute holding group membership. |

### LDAP / Active Directory

| Key | Default | Description |
|---|---|---|
| `mium.sso.ldap.enabled` | `false` | Enable the directory provider. With it on, a directory password works on the ordinary login form as well as at `/auth/sso/ldap/login`. |
| `mium.sso.ldap.url` | `ldap://ldap.example.com:389` | Directory URL. Use `ldaps://` or enable StartTLS — otherwise the bind password crosses the network in the clear. |
| `mium.sso.ldap.bind.dn` | (empty) | Service account that searches for user entries. Authentication is search then bind: the user's DN cannot be constructed, since Active Directory puts people under `CN=John Doe,OU=Staff,…` where neither component is the login name. |
| `mium.sso.ldap.bind.password` | (empty) | Service account password. Credential. |
| `mium.sso.ldap.user.base.dn` | (empty) | Subtree searched for user entries. |
| `mium.sso.ldap.user.filter` | `(uid={0})` | Filter locating the user; `{0}` is the login name, escaped per RFC 4515 before substitution. Active Directory usually wants `(sAMAccountName={0})`. |
| `mium.sso.ldap.group.base.dn` | (empty) | Subtree searched for groups. |
| `mium.sso.ldap.group.filter` | `(member={0})` | Filter locating groups containing the user; `{0}` is the user's DN. Membership is read both from this search **and** from the user's `memberOf`, because directories disagree about which side records it. |
| `mium.sso.ldap.group.name.attribute` | `cn` | Attribute holding the group name. |
| `mium.sso.ldap.starttls` | `false` | Upgrade a plain `ldap://` connection with StartTLS. |

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
| `mium.retention.sweep.interval.seconds` | `3600` | How often the leader runs the retention sweep. Paused entirely while the cluster is in [maintenance mode](../features/cluster-maintenance.md), so the sweep cannot delete data a restore has just brought back. |
| `mium.cluster.maintenance.retry.after.seconds` | `30` | How long a client is told to wait (`Retry-After`, seconds) before retrying a write that a maintenance window refused. The window itself is a runtime switch stored in the `mium_settings` table, not a property &mdash; toggle it from **Nodes** in the Admin UI or `POST /api/admin/maintenance`. |

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
