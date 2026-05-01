# Prerequisites

Before installing Mium you need a small set of external services that the platform depends on. Mium is the AI control plane for the CCL stack — it stores its own bootstrap state in embedded RocksDB, but everything above the bootstrap layer (chat history, prompts, embeddings, exported files) lives in shared services that must be reachable from every Master and Worker.

## Runtime

| Component | Version | Notes |
|---|---|---|
| Java | 17 or later | Required by Master and Worker. Detected via `JAVA_HOME` or `PATH`. |
| Python | 3.9 or later | Required on every **Worker** for server-side export rendering. The Python wheels (`openpyxl`, `reportlab`, `python-pptx`) are pre-bundled under `lib/python/` in the distribution — no `pip install` step at install time. |

## Required External Services

Mium will refuse to come up if any of these are missing.

| Service | Purpose | Default Endpoint |
|---|---|---|
| **ZooKeeper** | Leader election, service discovery, cluster readiness flag. | `localhost:2181` |
| **NeorunBase** (PostgreSQL-compatible) | Memory, Prompt, and Embedding stores. Tables auto-created on startup. | `jdbc:postgresql://localhost:5432/neorunbase` |
| **S3-compatible object store** | Server-rendered export ciphertext. Works with ShannonStore, MinIO, or AWS S3. | `http://localhost:8000` (default for ShannonStore) |
| **Ontul** | The data engine Mium queries on the user's behalf. Mium without Ontul can boot, but it has nothing to query — every shipped capability is built around Ontul as the data source. | per-user, registered in the ConnectionStore |

## Required Environment

| Variable | Required on | Purpose |
|---|---|---|
| `MIUM_MASTER_KEY` | Every Master and Worker | 32+ character secret used to derive the KMS root key (PBKDF2-SHA256 / 200K iterations). Must be **identical** on every node in the cluster — followers cannot decrypt the leader's KMS bundle without it. |
| `JAVA_HOME` | Every Master and Worker | Optional if `java` is on `PATH`; required otherwise. |
| `MIUM_HOME` | Every Master and Worker | Set automatically by the launcher scripts to the package root. Override only if you run Mium from a non-standard layout. |

Generate a master key once and store it in your secret manager:

```bash
openssl rand -base64 48 | head -c 32
```

## Optional LLM Backends

Each user brings their own LLM credentials via the ConnectionStore. The platform itself does not require an LLM at install time, but at least one of the following must be reachable for end-to-end chat to work:

- **Anthropic Claude** — Messages API. Connection `tool=anthropic`, `authType=api-key`.
- **Ollama** — local or self-hosted. Connection `tool=ollama`, `authType=api-key` (optional).

Embeddings used by retrieval features are served by `OllamaEmbeddingBackend` (e.g. `bge-m3`).

## Network

| Port | Direction | Purpose |
|---|---|---|
| `8090` | Inbound (browser, REST clients) | Master HTTP — Admin UI and REST API |
| `19099` | Master ↔ Master, Worker → Master | Master internal NIO (control plane) |
| `19098` | Master → Worker | Worker internal NIO (execute agent / tool / export) |
| `2181` | All nodes → ZooKeeper | Curator client traffic |
| `5432` | All nodes → NeorunBase | JDBC traffic |
| S3 endpoint | Master + Worker → S3 | Temp-file ciphertext upload / download |

The browser↔Master HTTP path is expected to terminate TLS at your reverse proxy (nginx, Envoy, etc.). Mium does not terminate TLS itself.

## Sizing Hints

These are starting points, not hard requirements.

| Role | Heap | Disk | CPU |
|---|---|---|---|
| Master | 2–4 GB | 20 GB (RocksDB stores) | 2–4 cores |
| Worker | 4–8 GB | 5 GB (logs + temp render scratch) | 4+ cores |

Workers are stateless beyond local logs — scale them out for parallel tool / LLM / export load. Masters scale for HA, not throughput.
