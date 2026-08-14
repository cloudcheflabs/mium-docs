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
- **The semantic layer** — semantic views and their metrics, dimensions and join
  paths; metric search that resolves plain English ("revenue") to a metric name;
  retrievers; and the ontology's object and link types. Read over Ontul's REST API
  under the user's own Ontul credentials, and injected into the system prompt
  ahead of the raw tables. This is what the model is aimed at; see
  [Grounded Answers](grounded-answers.md)
- Schema browsing (`SHOW CATALOGS`, `SHOW SCHEMAS`, `SHOW TABLES`, `DESCRIBE`)
- Catalog management — `list_catalogs`, `register_catalog`, `unregister_catalog`, plus a generic `ontul_admin` passthrough to the Ontul Admin REST API
- Full job lifecycle — submit batch / streaming jobs, poll status, stream logs, kill, list active and historical jobs (see [Job Lifecycle](job-lifecycle.md)). Job types are the `OntulJobType` enum values `BATCH`, `STREAMING`, `CLASS`, `PYTHON`
- Dependency upload for `CLASS` / `PYTHON` jobs (`uploadDep`, `listServerDeps`, auto-deps)
- Retrieval-grounded code generation (see below)

### Code Generation

`generate_code` produces Ontul SDK source grounded in real few-shot examples and the connected schema:

- A curated example index (`OntulExampleIndex`) holds Ontul SDK samples (`batch-dataframe.java`, `streaming-windowed.java`, `class-submit.java`, `remote-query.java`, `python-map.py`, `remote-query.py`) bundled under `mium-tools/.../resources/ontul-examples/`.
- Retrieval is three-tier: persistent vector search (namespace `ontul-codegen-examples`), then in-memory embeddings, then a tag-overlap fallback — so codegen still works without an embedding backend.
- The router (`OntulRouteResult`) can classify a request as `CODE`, `CLARIFY` (ask the user a question first), or `META` (e.g. fall through to `list_jobs`), and a fix-it loop can repair generated code from a reported error (`/api/jobs/codegen-fix`).

## How a Tool Call Flows

1. When a chat session starts, the agent loop assembles the system prompt from
   Ontul's semantic definitions relevant to the question plus `sqlReference()`
   from the tools available to the user.
2. The LLM sees the tool descriptions and emits a strict-JSON action that names the SQL or job operation it wants to run.
3. The agent loop dispatches the call. For `query` actions, it executes against the connected Ontul cluster directly; for job actions it calls the matching Ontul Admin REST endpoint.
4. The agent loop can offload entire LLM iterations to a Worker via `EXECUTE_AGENT` for parallelism, or dispatch only the tool call via `EXECUTE_TOOL`. Worker execution falls back to the master if no Worker is ready. Note the `EXECUTE_TOOL` worker path handles only `query` / `listTables` / `describeTable`; job-lifecycle actions always run on the master through the Ontul Admin REST client.
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
