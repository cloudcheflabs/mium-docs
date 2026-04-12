# Connection Management

Mium provides a per-user encrypted credential vault — the ConnectionStore — for managing connections to external systems (LLM providers, databases, APIs). Credentials are encrypted at rest via KMS and are never shared between users.

## ConnectionStore

The ConnectionStore is a RocksDB-backed store that holds per-user connection credentials:

- Each user manages their own set of connections
- Credentials are encrypted using KMS envelope encryption (AES-256-GCM)
- The store is replicated from the leader Master to followers for high availability

## Connection Types

### LLM Connections

Configure access to LLM providers:

- **Anthropic Claude**: API key
- **OpenAI**: API key
- **vLLM / Ollama**: Endpoint URL

### Tool Connections

Configure access to external tools:

- **Ontul**: Master endpoint, authentication token
- **Snowflake**: Account, username, password, warehouse
- **GitHub**: Personal access token or OAuth app credentials
- **Slack**: Bot token or OAuth credentials

## Management

Connections are managed through the Admin UI (Settings > Connections) or the REST API:

- **Create**: Add a new connection with provider type and credentials
- **Update**: Modify connection settings or rotate credentials
- **Delete**: Remove a connection
- **List**: View all configured connections for the current user

## Security

- Credentials are encrypted at rest using KMS envelope encryption
- Each connection's auth data is encrypted with its own Data Encryption Key (DEK)
- DEKs are encrypted with the master key
- Connection IDs are used throughout the system — raw credentials never appear in logs, configs, or API responses
- The ConnectionStore is synchronized from the leader to follower Masters via the NIO protocol
