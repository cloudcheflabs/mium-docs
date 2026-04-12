# Tool Integration

Mium treats every external system as a **Tool** — a pluggable component that the LLM can invoke during a conversation. The Tool SPI provides a clean abstraction for exposing any system with an API to the AI agent.

## Tool SPI

The `Tool` interface defines how external systems are exposed to the LLM:

- **Metadata**: Tool name, description, and capabilities
- **SQL Reference**: A structured description injected into the LLM system prompt so the model understands what the tool can do and how to invoke it
- **Execution**: The actual logic for calling the external system with user-provided credentials

## Built-in Tools

### Ontul SQL

Query the Ontul distributed data engine via SQL. The LLM can:

- Execute SQL queries against any registered Ontul catalog
- Browse schemas, tables, and columns
- Run analytical queries and return results in the chat

## How Tools Work

1. When a chat session starts, Mium assembles the system prompt by collecting `sqlReference()` from all tools the user has configured connections for
2. The LLM sees the tool descriptions and knows what external capabilities are available
3. When the user asks a question that requires data from an external system, the LLM generates a tool invocation in its JSON response
4. The agent loop dispatches the tool call to a Worker, which retrieves the user's credentials from the ConnectionStore and executes the operation
5. Results are returned to the LLM for interpretation and response generation

## Per-User Credential Isolation

Tools use credentials from the user's ConnectionStore entry. This means:

- Each user configures their own access to external systems
- Credentials are encrypted at rest via KMS
- The tool's own access control decides what data the user can access
- Mium does NOT federate IAM with external systems — no principal pass-through complexity

## Adding New Tools

New tools are implemented by:

1. Implementing the `Tool` interface
2. Providing a `sqlReference()` that describes the tool's capabilities for the LLM
3. Registering the tool in the tool registry

The tool system is designed to be extensible — any system with an API can become a Mium tool.
