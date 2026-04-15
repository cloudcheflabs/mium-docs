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

Core cluster state (IAM, KMS, ConnectionStore) is stored in embedded RocksDB on every Master — no external database needed for the platform to start. Memory, prompt, and embedding stores additionally support NeorunBase as the shared CCL-stack database when Mium is deployed alongside it. See [Storage Backends](storage-backends.md) for the full matrix.

## Snapshot Replication

The leader publishes a version-bumped snapshot to followers on every successful write. Followers pull the full snapshot when they see a version they haven't caught up to, and self-heal by periodically re-pulling. The write-only-on-leader invariant means mutating requests that land on a follower are transparently proxied to the leader via `LeaderRouter`.
