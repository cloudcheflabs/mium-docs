# High Availability

Mium provides fault tolerance and high availability through multi-Master leader election, state replication, and automatic failure recovery.

## Master High Availability

Multiple Masters can run simultaneously in a leader/follower configuration:

- **Leader Election**: Apache ZooKeeper (via Curator LeaderLatch) elects a primary Master. The leader owns all write operations to the state stores (RocksDB).
- **State Replication**: The leader Master replicates IAM, KMS, ConnectionStore, and MemoryStore state to follower Masters via the internal NIO protocol.
- **Automatic Failover**: If the leader Master fails, ZooKeeper elects a new leader, which reloads persisted state from RocksDB and resumes operations.
- **Transparent Proxying**: Follower Masters can serve read requests. Write requests received by followers are transparently proxied to the leader via `LeaderRouter`.
- **Self-Healing**: Followers periodically pull state snapshots from the leader to ensure consistency.

## Worker Registration

Workers register as ephemeral nodes in ZooKeeper under `/mium/workers/<nodeId>`:

- Automatic detection of worker joins and failures
- When a Worker disconnects, its ephemeral node is automatically removed
- The Master adjusts tool execution routing based on available Workers

## Service Discovery

Masters and Workers register as ephemeral nodes in ZooKeeper. When a node joins or leaves the cluster, all other nodes are notified automatically. No manual configuration of cluster membership is required.

ZooKeeper node layout:

```
/mium/
  masters/
    <nodeId-1>  (ephemeral)
    <nodeId-2>  (ephemeral)
  workers/
    <nodeId-3>  (ephemeral)
    <nodeId-4>  (ephemeral)
```

## State Stores

All cluster state is stored in embedded RocksDB — no external database is needed:

- IAM users, groups, policies, and organizations
- KMS encrypted keystore with versioned KEKs
- Per-user connection credentials (encrypted)
- Per-user chat sessions and messages

## State Synchronization OpCodes

| OpCode | Purpose |
|--------|---------|
| IAM_SYNC | User, group, policy, organization data |
| KMS_SYNC | Encryption key bundle |
| CONNECTION_SYNC | Per-user tool credentials |
| MEMORY_SYNC | Chat sessions and messages |
