# Admin UI

Mium ships a built-in React web application for chatting with LLMs, managing per-user settings, and administering the cluster. The UI is served by the master at `/admin/`.

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

### Settings — Your Account

- **Profile** — display name, contact, locale.
- **LLM Backends** — register Anthropic Claude or Ollama connections.
- **Tool Connections** — register Ontul (Arrow Flight access keys) and S3-compatible (ShannonStore / MinIO / AWS S3) connections.
- **Password** — change your password.

### Settings — Administration (admin-only)

- **Dashboard** — cluster metrics and health (CPU, heap, thread count per node) plus worker status, charted with Recharts.
- **Topology** — live registry of active Masters and Workers with health flags.
- **IAM** — users, groups, policies, companies, organizations; visual policy editor; access-key issuance.
- **Security & KMS** — list and rotate envelope-encryption keys.
- **Temp File Storage** — switch the server-side export ciphertext backend between local filesystem and S3-compatible object storage; "Test connection" probe.

## REST API

Every operation in the UI is also reachable on the Admin HTTP server (default port 8090). Endpoints are mounted under `/admin/api/*`:

- **Auth** — `/admin/auth/login`, `/admin/auth/refresh`, `/admin/auth/change-password`, `/admin/auth/whoami`
- **IAM** — `/admin/api/iam/users`, `/admin/api/iam/groups`, `/admin/api/iam/policies`, `/admin/api/iam/companies`, `/admin/api/iam/orgs`, `/admin/api/iam/keys`
- **STS** — `/admin/api/sts/sessions`, `/admin/api/sts/assume-role`
- **Connections** — `/admin/api/connections`
- **KMS** — `/admin/api/kms/list`, `/admin/api/kms/status/{keyId}`, `/admin/api/kms/rotate/{keyId}`
- **Chat** — `/admin/api/chat`, `/admin/api/chat/sessions`
- **Server-side Export** — `/admin/api/export`
- **Temp File Settings** — `/admin/api/settings/tempfile`
- **Monitoring** — `/admin/api/nodes/masters`, `/admin/api/nodes/workers`, `/admin/api/monitoring/metrics`, `/admin/api/logs/tail`

## Metrics & Observability

Mium collects per-node JVM metrics (CPU, heap, threads) and aggregates them centrally for the Dashboard. Workers report metrics to the Master via the internal NIO protocol. Logs from any node are tailable in real time, no SSH required.
