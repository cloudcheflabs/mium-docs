# Admin UI

Mium includes a built-in React web application for chatting with LLMs, managing settings, and administering the cluster.

## Technology Stack

- React 18 with TypeScript
- Vite build system
- Tailwind CSS for styling
- Recharts for metrics visualization
- React Router DOM for navigation

## Pages

### Chat (Default Home)

The primary interface for interacting with LLMs:

- Conversational chat with tool-calling capabilities
- Create and manage multiple chat sessions
- Resume previous conversations with full context
- Export conversations to PDF, PPTX, or XLSX

Routes: `/`, `/c/:sessionId`

### Settings

User-level configuration pages:

- **Profile** (`/settings/profile`): User profile management
- **LLM** (`/settings/llm`): Configure LLM provider connection (API key, model selection)
- **Connections** (`/settings/connections`): Manage tool connections (Ontul, Snowflake, GitHub, Slack)
- **Password** (`/settings/password`): Change password

### Admin Pages (Admin-Only)

Administrative pages visible only to admin users:

- **Dashboard** (`/settings/admin/dashboard`): Cluster metrics and health — query throughput, latency, JVM heap usage, and worker status visualized with Recharts
- **Topology** (`/settings/admin/topology`): Node registry and cluster topology — active Masters and Workers with node status
- **IAM** (`/settings/admin/iam`): User, group, policy, and organization management with a visual policy editor
- **KMS** (`/settings/admin/kms`): Key management interface for viewing and managing encryption keys

## REST API

All operations available in the Admin UI are also accessible via the REST API on the Admin HTTP server (default port 8090):

- **Auth**: `/admin/auth/login`, `/admin/auth/logout`, `/admin/auth/refresh`, `/admin/auth/password`
- **IAM**: `/admin/iam/users`, `/admin/iam/groups`, `/admin/iam/policies`, `/admin/iam/organizations`
- **Connections**: `/admin/connections`
- **KMS**: `/admin/kms`
- **Chat**: `/admin/chat/sessions`, `/admin/chat/messages`
- **Monitoring**: `/admin/monitoring/topology`, `/admin/monitoring/metrics`, `/admin/monitoring/logs`

## Metrics

Mium collects JVM and cluster metrics from all nodes:

- CPU usage, heap usage, thread count (per node)
- Node health and availability
- Cluster-wide aggregated metrics on the Dashboard

Metrics are collected from Workers via the NIO protocol (METRICS_REQ opcode) and visualized in the Dashboard.

## Real-Time Log Streaming

The Admin UI supports real-time log tailing from any node in the cluster, providing observability into Master and Worker operations without SSH access.
