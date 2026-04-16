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

IAM, KMS, and ConnectionStore are stored in embedded RocksDB on every Master — these bootstrap the cluster and cannot depend on an external database. Memory, Prompt, and Embedding stores live in NeorunBase; TempFile stores live in S3-compatible storage. See [Storage Backends](storage-backends.md).

## Snapshot Replication

Only IAM, KMS, and ConnectionStore are replicated between Mium nodes. The leader pushes snapshots to followers on every successful write. Followers self-heal by periodically re-pulling. Memory, Prompt, and Embedding data live in NeorunBase and do not require Mium-side replication.

## Cluster Readiness

The leader does not accept user requests until every registered node is ready:

1. Leader seeds KMS keys and sets `leader-ready` in ZK.
2. Non-leader Masters and Workers watch for `leader-ready`, then pull KMS + IAM + ConnectionStore from the leader.
3. Each node marks itself `ready=true` in ZK after all three stores are synced.
4. The leader polls ZK every 2 seconds. Only when all non-leader Masters and all Workers are `ready=true` does the leader start accepting requests.
5. If any node becomes unready (e.g. a Worker disconnects), the leader stops accepting requests until the cluster is whole again.
