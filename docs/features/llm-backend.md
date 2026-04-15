# LLM Backend

Mium decouples from any single LLM vendor through a pluggable `LlmBackend` interface. Users connect their own LLM provider via the ConnectionStore, and the agent loop interacts with the LLM through a provider-agnostic protocol.

## Pluggable Architecture

The `LlmBackend` interface abstracts all LLM interactions:

- **`AnthropicLlmBackend`** — Anthropic Claude (Messages API).
- **`OllamaLlmBackend`** — local or self-hosted Ollama; supports any model that Ollama serves.
- **`LlmBackendFactory`** — builds a backend instance from a stored connection (`tool=anthropic` or `tool=ollama`), pulling the API key / endpoint out of the user's encrypted ConnectionStore entry.

Each user can configure their own LLM provider — one user might use Claude while another points at a self-hosted Ollama instance.

## Strict JSON Protocol

Mium does not use vendor-specific tool APIs (Anthropic `tool_use` or OpenAI function-calling). The agent loop instead instructs the LLM to respond with a strict JSON object describing the next action — `query`, `submit_batch`, `submit_streaming`, `job_status`, `job_logs`, `kill_job`, `list_jobs`, `list_history`, or `generate_code` (see [Job Lifecycle](job-lifecycle.md)). This keeps the agent loop and the tool layer portable across providers; switching providers is a matter of registering a different connection.

## Embeddings

For retrieval-augmented features (e.g. few-shot pools used by code generation), Mium uses an embedding backend separate from the chat backend. The shipped implementation is `OllamaEmbeddingBackend`; vector storage is handled by the [Storage Backends](storage-backends.md) layer.

## Configuration

Connections are managed via the Admin UI (Settings → LLM Backends) or the REST API. Each connection stores:

- Tool type (`anthropic` or `ollama`)
- Endpoint URL
- API key (envelope-encrypted in the ConnectionStore)
- Model name in metadata

The connection is used automatically by the agent loop for all chat sessions owned by that user.
