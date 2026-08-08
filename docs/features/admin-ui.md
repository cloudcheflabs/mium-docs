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

A data-analysis workspace with a catalog tree browser (backed by `/api/catalog/connections`, `/api/catalog/tables`, `/api/catalog/columns`) alongside the chat panel. Routes: `/analyze`, `/analyze/c/:sessionId`.

### Dev workspace

A notebook-style workspace for generating and running Ontul jobs and remote Java/Python code cells, backed by the `/api/jobs/*` REST surface (`submit`, `list`, `status`, `logs`, `kill`, `codegen`, `codegen-fix`). Routes: `/dev`, `/dev/c/:sessionId`. See [Job Lifecycle](job-lifecycle.md).

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
- **Backup** — `/api/backup/config`, `/api/backup/run`, `/api/backup/history`, `/api/backup/list`, `/api/backup/restore`
- **Monitoring** — `/api/nodes/masters`, `/api/nodes/workers`, `/api/leader`, `/api/monitoring/metrics`, `/api/logs/tail`

## Metrics & Observability

Mium collects per-node JVM metrics (CPU, heap, threads) and aggregates them centrally for the Dashboard. The Master **polls** each node's JVM metrics over the internal NIO protocol (a `METRICS_REQ` pull), rather than nodes pushing. Logs from any node are tailable in real time, no SSH required.
