# Storage Backends

Mium splits its persistent state across a small set of stores, each with one or more interchangeable backend implementations. The split lets operators run Mium standalone (everything on local RocksDB) or deploy it alongside the rest of the CCL stack and share the heavyweight stores with NeorunBase.

## Store / Backend Matrix

| Store | Backends | Default | Config Key |
|---|---|---|---|
| IAM | RocksDB | RocksDB | `mium.iam.rocksdb.path` |
| KMS | RocksDB | RocksDB | `mium.kms.rocksdb.path` |
| Connection | RocksDB | RocksDB | `mium.connection.rocksdb.path` |
| Memory (chat) | RocksDB / NeorunBase | RocksDB | `mium.memory.backend` |
| Prompt | RocksDB / NeorunBase | RocksDB | `mium.prompt.backend` |
| Embedding | In-memory / NeorunBase | In-memory | `mium.embedding.store` |
| Temp File | Local FS / S3-compatible | Local FS | `mium.tempfile.backend` |

IAM, KMS, and ConnectionStore are intentionally pinned to RocksDB — they bootstrap the cluster itself and can't depend on a database that hasn't started yet.

## RocksDB Backends

The historical default. Each store owns an envelope-encrypted RocksDB blob on the leader Master. Followers receive snapshot pushes via internal NIO sync (`MEMORY_SYNC`, `PROMPT_SYNC`, etc.) and serve reads from their local copy. Workers pull the connection / KMS slices they need to decrypt user credentials.

Operationally simple — a single packaged Mium dist starts with no external database. Snapshots live next to the cluster's master key, so backup is straightforward.

## NeorunBase Backends

When Mium is deployed alongside NeorunBase, Memory / Prompt / Embedding can be delegated to it:

- The corresponding `mium_chat_session`, `mium_chat_message`, `mium_prompt`, `mium_embedding` tables are auto-created in the configured schema.
- Every Mium node reads and writes live — there is no Mium-side replication, because NeorunBase handles durability and consistency.
- Snapshot opcodes (`MEMORY_SYNC`, `PROMPT_SYNC`) are no-ops in this configuration.

NeorunBase access is configured by:

```properties
mium.memory.backend          = neorunbase
mium.prompt.backend          = neorunbase
mium.neorunbase.jdbc.url     = jdbc:postgresql://<host>:5432/neorunbase?preferQueryMode=simple
mium.neorunbase.username     = admin
mium.neorunbase.password     = ...
mium.neorunbase.schema       = mium
```

## Embedding Store

The shipped default is the in-memory cosine store (`InMemoryEmbeddingStore`) which is appropriate for the few-shot pools used by code generation today. The NeorunBase backend lives behind `mium.embedding.store=neorunbase` and uses `VECTOR(N)` columns plus `<=>` cosine distance ordering once the deployment's NeorunBase has the vector capability shipped.

## Temp File Store

See [Temp File Storage](temp-file-storage.md). Default is local filesystem; switch to an S3-compatible backend via `mium.tempfile.backend=s3` and a `tool=s3` Connection.

## Choosing

The decision is binary per store:

- **Standalone or single-host demo** — keep all defaults (RocksDB everywhere, in-memory embeddings, local temp files).
- **CCL stack deployment** — flip Memory / Prompt / Embedding to NeorunBase and Temp File to S3-compatible (ShannonStore). Each Mium node becomes effectively stateless beyond its IAM / KMS / Connection footprint.
