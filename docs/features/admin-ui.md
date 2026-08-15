# Admin UI

Mium ships a built-in React web application for chatting with LLMs, managing per-user settings, and administering the cluster. By default the UI is served by the master at the **root path `/`**, with the REST API under `/api/*` and auth under `/auth/*`. The whole surface is re-prefixed only when `mium.admin.context.path` is set (e.g. to `/admin`, in which case the Vite bundle must be rebuilt with `VITE_BASE_PATH=/admin`).

## Technology Stack

- React 18 with TypeScript
- Vite build system
- Tailwind CSS
- Recharts for metrics visualization
- React Router DOM

## Pages

### Chat (default home)

The primary interface for interacting with LLMs:

- Conversational chat with tool-calling, chart rendering, and result-payload export buttons (CSV / Markdown / XLSX / PDF / PPTX rendered client-side).
- Create, rename, and delete chat sessions; resume any past conversation with full context.

Routes: `/`, `/c/:sessionId`. The Settings entry point lives in the user footer at the bottom-left of the chat sidebar.

### Analyze workspace

The workspace for asking questions of data. Routes: `/analyze`,
`/analyze/c/:sessionId`.

It is **answer-first**: the assistant's conclusion is the primary output, with a
trust bar beneath it saying which semantic views, ontology entities and metrics
the answer used and whether Ontul certifies them — with *Certification out of
date* and Ontul's own reason when a definition has changed since it was signed
off (see [Grounded Answers](grounded-answers.md)). The
SQL is one click away and stays editable and runnable — an analyst who wants to
take over should not have to leave the page — but it is the receipt, not the
product. A single-value result is rendered as a headline figure rather than a
one-cell table.

Alongside the answer:

- **Definition browser** on the left, open by default: semantic views, metrics,
  entities, relationships and retrievers, read from `/api/semantic/*`. A user who
  cannot see what is defined has to guess what the data covers.
- **Starter questions**, generated from the certified definitions, so every
  opening suggestion has an answer waiting rather than pointing at a raw table
  the model is told not to use.
- **Follow-ups**, derived from the columns that actually came back, so no
  suggestion leads to a dead end.
- **Was this right?** beneath every answer — one click to report a wrong answer,
  and for curators, one click to verify a correct one. See
  [Verified Answers](verified-answers.md).

The raw catalog tree (`/api/catalog/connections`, `/api/catalog/tables`,
`/api/catalog/columns`) remains available for browsing physical tables.

### Dev workspace

A notebook-style workspace for generating and running Ontul jobs and remote
Java/Python code cells, backed by the `/api/jobs/*` REST surface (`submit`,
`list`, `status`, `logs`, `kill`, `codegen`, `codegen-fix`). Routes: `/dev`,
`/dev/c/:sessionId`. See [Job Lifecycle](job-lifecycle.md).

Each prompt is echoed as an **Asked** block above its result, so a session reads
as a transcript rather than replacing itself; every past question can be copied
or reused with one click, and the prompt box sits below the results where the
next question is written.

### Settings — Your Account

- **Profile** — display name, contact, locale.
- **LLM Backends** — register Anthropic Claude or Ollama connections.
- **Tool Connections** — register Ontul (Arrow Flight access keys) and S3-compatible (ShannonStore / MinIO / AWS S3) connections.
- **Storage** — view and wipe your own stored data (memory / prompt / embedding), backed by `DELETE /api/storage/me`.
- **Password** — change your password.

### Settings — Administration (admin-only)

- **Dashboard** — cluster metrics and health (CPU, heap, thread count per node) plus worker status, charted with Recharts.
- **Topology** — live registry of active Masters and Workers with health flags.
- **IAM** — users, groups, policies, companies, organizations; visual policy editor; access-key issuance.
- **Security & KMS** — list, create, and rotate envelope-encryption keys.
- **Temp File Storage** — configure the S3-compatible object store used for server-side export ciphertext (endpoint, region, bucket, path-style, ConnectionStore id); "Test connection" probe (`/api/settings/tempfile/test`).
- **Embedding** — configure the embedding backend/models (`/api/settings/embedding`).
- **Verified Answers** — the review queue for reported answers and the library of
  verified question/SQL pairs. See [Verified Answers](verified-answers.md).
- **Instructions** — plain-language house rules added to every prompt, workspace-wide
  and per connection. See [Instructions](instructions.md).
- **Benchmarks** — questions with a known-correct answer, run on demand and scored,
  with the history of past runs. See [Benchmarks](benchmarks.md).
- **Backup & Restore** — configure and run backups, browse and restore snapshots
  (`/api/backup/*`). See [Backup & Restore](backup-restore.md).

## REST API

Every operation in the UI is also reachable on the Admin HTTP server (default port 8090). By default endpoints are mounted under `/api/*` with auth under `/auth/*` (a `/admin` prefix applies only when `mium.admin.context.path` is set):

- **Auth** — `/auth/login`, `/auth/refresh`, `/auth/change-password`, `/auth/whoami`
- **IAM** — `/api/iam/users`, `/api/iam/users/me`, `/api/iam/groups`, `/api/iam/policies`, `/api/iam/add-user-to-group`, `/api/iam/remove-user-from-group`, `/api/iam/attach-group-policy`, `/api/iam/detach-group-policy`, `/api/iam/keys`, `/api/iam/download-key`, `/api/iam/companies` (read-only), `/api/iam/orgs` (read-only)
- **STS** — `/api/sts/sessions`, `/api/sts/assume-role`, `/api/sts/sessions/clear`, `/api/sts/sessions/download`
- **Connections** — `/api/connections`
- **KMS** — `/api/kms/list`, `/api/kms/status/{keyId}`, `/api/kms/create`, `/api/kms/rotate/{keyId}`
- **Chat** — `/api/chat`, `/api/chat/sessions`, `/api/chat/sessions/all`, `/api/chat/sessions/messages`, `/api/chat/sessions/rename`
- **Catalog** — `/api/catalog/connections`, `/api/catalog/tables`, `/api/catalog/columns`
- **Jobs** — `/api/jobs/submit`, `/api/jobs/status`, `/api/jobs/logs`, `/api/jobs/list`, `/api/jobs/kill`, `/api/jobs/codegen`, `/api/jobs/codegen-fix`
- **Server-side Export** — `/api/export`, `/api/export/download`
- **Storage** — `/api/storage/me`, `/api/storage/purge`
- **Settings** — `/api/settings/tempfile`, `/api/settings/tempfile/test`, `/api/settings/embedding`
- **Semantic layer** — `/api/semantic/views`, `/api/semantic/views/detail`, `/api/semantic/metrics/search`, `/api/semantic/object-types`, `/api/semantic/link-types`, `/api/semantic/retrievers` (read-through to Ontul under the user's credentials)
- **Verified answers** — `/api/verified/report`, `/api/verified/verify`, `/api/verified`, `DELETE /api/verified`
- **Instructions** — `/api/instructions`, `/api/instructions/all`, `PUT /api/instructions`
- **Benchmarks** — `/api/benchmarks`, `/api/benchmarks/run`, `/api/benchmarks/runs`
- **Backup** — `/api/backup/config`, `/api/backup/run`, `/api/backup/history`, `/api/backup/list`, `/api/backup/restore`
- **Monitoring** — `/api/nodes/masters`, `/api/nodes/workers`, `/api/leader`, `/api/monitoring/metrics`, `/api/logs/tail`

## Metrics & Observability

Mium collects per-node JVM metrics (CPU, heap, threads) and aggregates them centrally for the Dashboard. The Master **polls** each node's JVM metrics over the internal NIO protocol (a `METRICS_REQ` pull), rather than nodes pushing. Logs from any node are tailable in real time, no SSH required.
