# Encryption & KMS

Mium provides built-in encryption with a Key Management Service (KMS) to protect sensitive data at rest across the cluster.

## Envelope Encryption

Mium uses envelope encryption — each piece of sensitive data is encrypted with its own Data Encryption Key (DEK), and DEKs are encrypted with the master key:

- **Algorithm**: AES-256-GCM
- **Master Key Derivation**: PBKDF2-SHA256 with 200,000 iterations
- **Master Key Source**: Environment variable specified by `mium.kms.master.key.env` (minimum 32 characters)
- **Versioned KEKs**: Key Encryption Keys are versioned, allowing key rotation without re-encrypting all data

## What Is Encrypted

- **Connection credentials**: LLM API keys, tool credentials (database passwords, API tokens) stored in the ConnectionStore
- **IAM secrets**: User passwords and authentication tokens
- **Cluster state metadata**: Sensitive metadata in the RocksDB state stores

## Built-in KMS

Mium includes a built-in KMS (MiumKmsProvider) that:

- Generates and manages encryption keys in a RocksDB-backed keystore
- Supports versioned Key Encryption Keys for rotation
- Distributes keys from the leader Master to all cluster nodes automatically
- Replicates the encrypted keystore to follower Masters for high availability

No external KMS service is required.

## Key Distribution

1. The leader Master generates and stores DEKs in the local encrypted RocksDB keystore
2. On leader election or key changes, the keystore is replicated to follower Masters via the internal NIO protocol (KMS_SYNC opcode)
3. Workers receive relevant keys for decrypting connection credentials needed during tool execution

## Configuration

```properties
# Enable KMS (required in production)
mium.kms.enabled=true

# Environment variable containing the master key
mium.kms.master.key.env=MIUM_MASTER_KEY

# RocksDB path for KMS keystore
mium.kms.rocksdb.path=/data/mium/kms
```

## Management

KMS keys are managed through the Admin UI (Settings > Admin > KMS) or the REST API.
