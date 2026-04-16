# Multi-Agent AI Platform

Mium is the AI agent platform for Ontul — an on-prem-first multi-agent system that lets users interact with Ontul through natural language. Without Ontul, Mium has nothing to query, no jobs to submit, and no code to generate. Every capability is built around Ontul as the data engine.

## Why Mium

Organisations running Ontul need a way to let non-SQL users query and operate on their data through natural language, without exposing credentials and without cloud dependencies. Mium addresses this with:

- A **credential vault** (ConnectionStore) where each user keeps their own tool credentials, encrypted at rest.
- A **Tool SPI** that exposes any system with an API to the LLM.
- An **agent loop** that orchestrates a strict-JSON action protocol covering data queries, long-running jobs, and code generation.
- A **sovereign deployment** model — IAM, KMS, connections, chat memory, and prompt library all live on your infrastructure.

## Core Concepts

### Agent Loop

Mium's agent loop turns each user message into one or more LLM calls plus the tool actions the LLM emits. The LLM responds with a single JSON object naming the action; the loop dispatches it and feeds the result back if another iteration is needed. The shipped action vocabulary is:

| Action | Purpose |
|--------|---------|
| `query` | Ad-hoc SQL against the connected tool, optionally with a chart hint (`bar`, `line`, `pie`, `table`, …) and an export format hint |
| `submit_batch` / `submit_streaming` | Submit a long-running Ontul job |
| `job_status` / `job_logs` / `kill_job` | Operate on a running or finished job by id |
| `list_jobs` / `list_history` | Active and historical job lists |
| `generate_code` | Produce Ontul SDK source (Java BATCH / STREAMING / CLASS, Python) the user can copy |

See [Job Lifecycle](job-lifecycle.md) for the job-related actions in detail.

### Tools

Every external system is a Tool — a pluggable component the LLM can invoke. Each tool provides metadata and a SQL/API reference that gets injected into the LLM system prompt so the model knows what is available and how to use it.

The shipped tool today is **Ontul** (Cloud Chef Labs SQL engine). The Tool SPI is open — additional tools register without modifying the agent loop. See [Tool Integration](tool-integration.md).

### Connections

Each user manages their own connections to external systems. Credentials are stored encrypted in the ConnectionStore and are never shared between users. When the LLM invokes a tool, Mium retrieves the calling user's credentials for that tool — the tool's own access control decides what data the user can touch.

## How It Works

1. User sends a message via the Chat UI or `/admin/api/chat`.
2. The agent loop builds a system prompt that includes the available tool descriptions and the user's rolling chat context.
3. The loop calls the user's configured LLM (Anthropic Claude or Ollama) and parses the strict-JSON response.
4. If the action requires a tool call, Mium dispatches it — locally on the master or offloaded to a Worker via `EXECUTE_AGENT` / `EXECUTE_TOOL` — using the user's encrypted credentials.
5. Tool results feed back to the LLM (when more iterations are needed) or directly to the user.
6. The conversation is persisted in the MemoryStore so the next turn has the full context.

## Deployment Model

Mium is the AI agent for Ontul, deployed as part of the CCL stack:

- IAM, KMS, and ConnectionStore live in embedded RocksDB on each Master (bootstrap deps).
- Memory, Prompt, and Embedding stores live in **NeorunBase** (shared CCL-stack database).
- Server-rendered export files live in any **S3-compatible** object store (ShannonStore, MinIO, AWS S3, etc.).
- ZooKeeper handles leader election and service discovery.
- Users bring their own LLM API keys via the ConnectionStore. No data leaves your network unless you point the LLM connection at a hosted provider.
