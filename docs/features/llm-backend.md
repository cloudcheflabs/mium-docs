# LLM Backend

Mium decouples from any single LLM vendor through a pluggable `LlmBackend` interface. Users connect their own LLM provider via the ConnectionStore, and the agent loop interacts with the LLM through a provider-agnostic protocol.

## Pluggable Architecture

The `LlmBackend` interface abstracts all LLM interactions:

- **AnthropicLlmBackend**: Anthropic Claude implementation (default)
- **LlmBackendFactory**: Builds backend instances from stored connections

Each user can configure their own LLM provider — one user might use Claude while another uses a self-hosted vLLM instance.

## Strict JSON Protocol

Mium does not use vendor-specific tool APIs (like Anthropic's tool_use or OpenAI's function calling). Instead, all LLM interactions use structured JSON replies:

- **Request**: `LlmRequest` containing messages, system prompt (with tool descriptions), and configuration
- **Reply**: `LlmReply` parsed from the LLM's JSON response, containing either a text response or tool invocations
- **Messages**: `LlmMessage` — a clean, provider-agnostic message format

This design ensures portability across Claude, OpenAI, vLLM, Ollama, and any future provider without changing the agent loop or tool system.

## Supported Providers

| Provider | Status | Notes |
|----------|--------|-------|
| Anthropic Claude | Supported | Default backend |
| OpenAI | Planned | GPT-4, GPT-4o |
| vLLM | Planned | Self-hosted open models |
| Ollama | Planned | Local model execution |

## Configuration

Users configure their LLM connection via the Admin UI Settings or REST API:

- Select provider type
- Enter API key or endpoint URL
- Configure model parameters (temperature, max tokens, etc.)

The connection is stored encrypted in the ConnectionStore and used automatically for all chat sessions.
