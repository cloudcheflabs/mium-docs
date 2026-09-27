# HA Cluster

A production Mium deployment runs **multiple Masters** for leader/follower failover and **multiple Workers** for parallel LLM, tool, and export load. ZooKeeper is the only thing that elects the leader; Mium does the rest itself.

This page walks through provisioning a multi-host cluster. The single-host walkthrough is in [Run Mium](running.md).

## Topology

```
                ┌─────────────────────────────────┐
                │        ZooKeeper Ensemble       │
                │     (3 or 5 nodes, your own)    │
                └──────────────┬──────────────────┘
                               │
        ┌──────────────────────┴───────────────────────┐
        │                                              │
   ┌────▼─────┐    ┌──────────┐    ┌──────────┐  ┌────▼─────┐
   │ Master 1 │ ←→ │ Master 2 │ ←→ │ Master 3 │  │ Worker N │
   │ (leader) │    │(follower)│    │(follower)│  │          │
   └────┬─────┘    └──────────┘    └──────────┘  └──────────┘
        │
   ┌────▼────────────┐  ┌──────────────────┐  ┌──────────────┐
   │   NeorunBase    │  │ S3-compatible    │  │   Ontul      │
   │ (chat / prompt) │  │  (export ct)     │  │   (data)     │
   └─────────────────┘  └──────────────────┘  └──────────────┘
```

Recommended starting point:

- **3 Masters** — survives one-node failure with quorum to spare.
- **2+ Workers** — required for export to work (the Master will not render Python locally; if no Worker is ready, exports return `503`).
- **3 ZooKeeper nodes** — your existing CCL stack ensemble is reused; Mium does not need its own.

## Per-Node Install

Repeat the [Install](install.md) steps on every host:

1. Unpack the same `mium-VERSION.tar.gz`.
2. Set the **same** `MIUM_MASTER_KEY` everywhere — followers cannot decrypt the leader's KMS bundle without it.
3. Edit `conf/mium.properties` with the values appropriate to that host.

## Cluster Configuration

Every node in the cluster shares these settings:

```properties
# Same ZK ensemble on every node
mium.zk.connect = zk1.internal:2181,zk2.internal:2181,zk3.internal:2181
mium.zk.root    = /mium

# Same NeorunBase on every node
mium.neorunbase.jdbc.url  = jdbc:postgresql://nb.internal:5432/neorunbase?preferQueryMode=simple
mium.neorunbase.username  = mium
mium.neorunbase.password  = <secret>
mium.neorunbase.schema    = mium

# Same S3-compatible store on every node
mium.tempfile.s3.endpoint = https://s3.internal
mium.tempfile.s3.region   = us-east-1
mium.tempfile.s3.bucket   = mium-tempfile
```

Per-node settings (different on each host):

```properties
# Where this node LISTENS. 0.0.0.0 is every interface.
mium.master.host             = 0.0.0.0
mium.master.admin.port       = 8090
mium.master.internal.port    = 19099
mium.worker.host             = 0.0.0.0
mium.worker.internal.port    = 19098

# Where the OTHER nodes reach it. Different from the bind address: 0.0.0.0 says
# "listen on everything" and is not an address anything can dial. Left unset,
# this is resolved to the host's own hostname, falling back to its first
# non-loopback interface — correct for most deployments.
#mium.master.advertised.host = master1.internal
#mium.worker.advertised.host = worker1.internal
```

!!! warning "Set these when the automatic answer is wrong"
    Name them explicitly on a host with several NICs where the cluster talks over
    one of them, or where the other nodes' DNS resolves this host's name to a
    different address than the host itself does. Getting it wrong is visible: each
    node logs the address it advertises at startup (`Advertised : …`), and a node
    that could find no routable address says so at WARN and advertises
    `127.0.0.1` — which works only when it is the only node.


## Bring Up the Cluster

Order matters on the first boot — the leader has to seed KMS keys before followers can sync.

1. Start ZooKeeper (already running in your environment).
2. Start Master #1. Wait for `leader-ready` in the log.
3. Start Master #2 and #3 in any order. Each will pull KMS / IAM / ConnectionStore from the leader and report ready — and publish `ready=true` to ZooKeeper, which is what lets step 5 complete.
4. Start every Worker. Each pulls the same three stores **from the leader** — not from any ready Master, because a follower holds a replica and a Worker seeded from one would inherit that lag without being able to detect it. Workers find the leader from the `/mium/leader` znode the leader publishes.
5. The leader polls ZooKeeper and starts accepting traffic only when **every** registered Master and Worker reports `ready=true`.

`curl http://master1:8090/ready` returns 200 only after step 5 succeeds.

## Front the Cluster With a Load Balancer

The Admin UI and REST API on port 8090 are stateless once the JWT is issued. Operators typically front all three Masters with a Layer-7 load balancer:

- **Health probe**: `GET /ready` on every Master. Mark unhealthy on 503.
- **Sticky sessions**: not required — JWTs work on any Master.
- **Write requests on followers**: Mium's `LeaderRouter` transparently proxies write traffic to the leader, so the LB does **not** need to know which Master is the leader.
- **Single sign-on**: no special handling. A redirect flow is several requests and they land on different Masters; the login state is sealed with a key every node derives from `MIUM_MASTER_KEY`, and the session a login produces is recorded on the leader and replicated, so any Master resolves it. See [Single Sign-On](../features/sso.md).

### How fast a change reaches the other Masters

Under a second. The leader sends a change notice on every write and the followers pull the affected snapshot immediately; a 30-second self-heal poll exists underneath as a backstop, not as the normal path. If you observe a change on one Master taking tens of seconds to appear on another, the notices are not arriving — check that each node advertises a routable address, and that nothing is filtering `mium.master.internal.port` between them.

Terminate TLS at the load balancer — Mium does not terminate TLS itself.

## Failover Behaviour

- **Leader Master fails**: ZooKeeper expires its session; Curator elects a new leader; the new leader reloads its embedded RocksDB stores and resumes accepting writes. The bounded interruption is `mium.zk.session.timeout.ms` (default 30s).
- **Follower Master fails**: traffic shifts to the remaining Masters via the load balancer. The leader marks the cluster unready until the follower returns or is removed from ZK.
- **Worker fails**: its ephemeral ZK node disappears. The leader will not dispatch new work to it. The cluster becomes unready and rejects new requests until the remaining Workers report `ready=true` again — operators add Worker capacity ahead of failure to keep this transparent.

## Adding a Worker Online

1. Install Mium on the new host as above.
2. Confirm `MIUM_MASTER_KEY` matches the rest of the cluster.
3. `bin/start-worker.sh`.

The Worker registers in ZK, pulls KMS / IAM / ConnectionStore, and reports ready. The leader detects it, re-validates cluster readiness, and resumes accepting requests.

## Removing a Worker

```bash
bin/stop-worker.sh
```

Graceful stop: the Worker deregisters from ZK, the leader sees the node count drop, the cluster stays ready as long as at least one Worker remains.

To take a host out of rotation permanently, stop the Worker, then unpack and reinstall on a different host — there is no per-Worker state to migrate.

## Adding a Master Online

Same procedure as a Worker. The new Master joins as a follower, syncs from the leader, and stays in standby for failover.

To **promote** an existing follower to leader, simply stop the current leader. The next leader election picks one of the remaining Masters.

## Operational Reads

- [Monitoring](../operations/monitoring.md) — the Topology page in the Admin UI shows live Master / Worker registries with health flags.
- [Backup & Restore](../operations/backup-restore.md) — what to back up across the three storage tiers.
- [Troubleshooting](../operations/troubleshooting.md) — `cluster-not-ready` is the single most common operational issue.
