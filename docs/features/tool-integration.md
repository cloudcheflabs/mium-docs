# Tool Integration

Mium's primary and foundational tool is **Ontul** — the CCL stack's distributed data engine. The `Tool` SPI provides a clean abstraction, but today Ontul is the only shipped tool and the reason Mium exists. Without an Ontul connection, the agent has nothing to query.

## Tool SPI

The `Tool` interface defines how external systems are exposed to the LLM:

- **Metadata** — tool name and capabilities surfaced to the agent loop.
- **`sqlReference()`** — a structured description of the tool's SQL dialect, available catalogs, and conventions, injected into the LLM system prompt so the model can produce valid invocations without trial and error.
- **Execution** — the actual call into the external system, using credentials decrypted from the user's ConnectionStore entry.

## Built-in Tool

### Ontul

Mium ships with one built-in tool: **Ontul**, the Cloud Chef Labs SQL engine. The agent connects via Arrow Flight SQL using the user's access-key credentials and supports:

- SQL queries against any registered Ontul catalog
- Schema browsing (`SHOW CATALOGS`, `SHOW SCHEMAS`, `SHOW TABLES`, `DESCRIBE`)
- Full job lifecycle — submit batch / streaming jobs, poll status, stream logs, kill, list active and historical jobs (see [Job Lifecycle](job-lifecycle.md))
- Server-side code generation (Java BATCH / STREAMING / CLASS, Python) producing source the user can copy or submit

## How a Tool Call Flows

1. When a chat session starts, the agent loop assembles the system prompt by collecting `sqlReference()` from the tools available to the user.
2. The LLM sees the tool descriptions and emits a strict-JSON action that names the SQL or job operation it wants to run.
3. The agent loop dispatches the call. For `query` actions, it executes against the connected Ontul cluster directly; for job actions it calls the matching Ontul Admin REST endpoint.
4. The agent loop can offload entire LLM iterations to a Worker via `EXECUTE_AGENT` for parallelism, or dispatch only the tool call via `EXECUTE_TOOL`. Worker execution falls back to the master if no Worker is ready.
5. Results are returned to the LLM for interpretation and a user-facing response.

## Per-User Credential Isolation

Tools use credentials from the calling user's ConnectionStore entry, which means:

- Each user configures their own access to external systems.
- Credentials are envelope-encrypted at rest via KMS; raw values never appear in logs.
- The tool's own access control decides what data the user can touch — Mium does NOT federate IAM with external systems.

## Adding New Tools

New tools are implemented by:

1. Implementing the `Tool` interface.
2. Providing `sqlReference()` (or equivalent capability description) for the LLM system prompt.
3. Registering the tool so the agent loop can find it.
4. Defining a `tool=...` value in the ConnectionStore so users can register credentials.

The tool system is designed to be extensible — additional CCL stack tools and external systems can plug in without touching the agent loop.
