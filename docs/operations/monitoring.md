# Monitoring & Observability

Mium reports cluster health through a single Master endpoint and surfaces it on the Admin UI Dashboard. There is no Prometheus exporter, no separate metrics agent — the Master collects metrics from every node over the internal NIO protocol and serves them on the same HTTP port the UI uses.

## What Is Collected

Every Master runs a collector that, every `mium.metrics.collect.interval.seconds` (default 10 s),
takes a snapshot of itself and asks every other Master and every Worker for theirs over the
internal NIO plane (`METRICS_REQ`). Snapshots are kept in memory for
`mium.metrics.retention.seconds` (default 3600 s) and served on `GET /api/monitoring/metrics`
(`?startTs=&endTs=`, epoch milliseconds; default the last hour) for the Dashboard's Recharts charts.
Each entry is `{timestamp, nodeId, role, data}`:

| `data` key | Meaning |
|---|---|
| `system_cpu_usage` | Process CPU %, from `OperatingSystemMXBean.getProcessCpuLoad()` |
| `system_mem_used`, `system_mem_max` | JVM heap used / max, bytes |
| `mium_threads` | Live thread count |
| `mium_masters_total`, `mium_workers_total`, `mium_workers_ready` | Cluster membership as seen in ZooKeeper |
| `mium_chat_sessions_total`, `mium_chat_sessions_active` | Chat sessions |
| `mium_llm_calls_total`, `mium_llm_calls_failed`, `mium_llm_tokens_total` | LLM usage |
| `mium_tool_calls_total`, `mium_tool_calls_failed` | Tool (Ontul, S3, ...) calls |

The metric format mirrors Ontul's so that operators familiar with the broader CCL stack see a consistent shape across products.

## Topology Probes

| Endpoint | Purpose |
|---|---|
| `GET /api/nodes/masters` | Live list of Master nodes with leader flag and ready state. |
| `GET /api/nodes/workers` | Live list of Worker nodes with ready state. |
| `GET /api/nodes/zookeeper` | The ZooKeeper ensemble — every server in `mium.zk.connect` with its status, role and load — and the answering Master's own ZooKeeper session. See [ZooKeeper](#zookeeper). |
| `GET /api/leader` | Whether the answering Master is the leader (`isLeader`), and the leader's admin `leaderHost` / `leaderPort`. |

The **Cluster Topology** page in the Admin UI (Settings → Administration → Topology) is built
directly on these — Masters, Workers and ZooKeeper — so anything you see there can be scripted.

## ZooKeeper

ZooKeeper holds Mium's leader election and node registry, so a ZooKeeper problem is a Mium
problem: a lost quorum stops leader election and readiness. The Topology page shows the ensemble
next to the Masters and Workers.

For every server in `mium.zk.connect`, the Master that answers opens a short socket of its own and
sends ZooKeeper's `srvr` four-letter word. That is the one command ZooKeeper 3.5+ allows without
configuration. If you set `4lw.commands.whitelist` on your ensemble, keep `srvr` in it. Servers are
probed in parallel, each with a `mium.zk.topology.probe.timeout.ms` (default 2000 ms) connect and
read timeout, so one dead server delays the page by at most that much. The probe does not go
through the Master's own ZooKeeper session, so a server the session is not attached to is still
reported.

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://mium:8090/api/nodes/zookeeper
```

```json
{
  "connectString": "zk1:2181,zk2:2181,zk3:2181",
  "root": "/mium",
  "servers": [
    {"host": "zk1", "port": 2181, "status": "UP", "mode": "follower", "version": "3.9.2",
     "latencyMin": 0.0, "latencyAvg": 0.42, "latencyMax": 12.0, "received": 1234, "sent": 1240,
     "connections": 3, "outstanding": 0, "zxid": "0x200000005", "nodeCount": 45, "responseMs": 3},
    {"host": "zk2", "port": 2181, "status": "UP", "mode": "leader", "...": "..."},
    {"host": "zk3", "port": 2181, "status": "DOWN",
     "error": "UnknownHostException: zk3"}
  ],
  "client": {"connected": true, "state": "CONNECTED", "sessionId": "0x10000a1b2c30001",
             "sessionTimeoutMs": 30000, "connectedServer": "zk1/172.18.0.3:2181"}
}
```

| Field | Meaning |
|---|---|
| `status` | `UP` — the server answered `srvr`. `DOWN` — no answer within the timeout, connection refused, unknown host, or the server answered that it is **not currently serving requests** (it is running but has no quorum, so clients cannot use it). `error` says which. |
| `mode` | `leader`, `follower`, `observer` or `standalone`. `unknown` when the server is reachable but `srvr` is disabled by `4lw.commands.whitelist`. |
| `version` | ZooKeeper version. |
| `connections`, `outstanding` | Client connections, and requests queued on the server. |
| `latencyMin` / `latencyAvg` / `latencyMax` | Request latency in milliseconds since the server's stats were last reset. |
| `zxid` | Last committed transaction id. In a healthy ensemble the servers are within a few transactions of one another. |
| `nodeCount` | Number of znodes. |
| `client` | The answering Master's own Curator session: `state` (`CONNECTED`, `CONNECTING`, ...), session id, negotiated session timeout and the server it is attached to. Different Masters are usually attached to different servers. |

In the UI each server is a row with a role badge (the leader is marked with a crown), an
Online / Offline status, version, connections, latency, znode count and zxid. A DOWN server shows
its error under the address. The header shows how many servers are up, and a line under it
shows the answering Master's session.

## Health and Readiness Probes

| Endpoint | Purpose |
|---|---|
| `GET /health` | Liveness. Returns `200` once the JVM is up. Use for process-supervisor / Docker `HEALTHCHECK`. |
| `GET /ready` | Readiness. Returns `200` only when the cluster is whole — every Master and Worker has synced KMS / IAM / ConnectionStore. Returns `503` until then. Use for load-balancer health checks. |

A common mistake is to point the load balancer at `/health` — that traps traffic on a Master that is up but cannot serve writes (e.g. the leader is down). Always use `/ready` for LB health, `/health` for process supervisors.

## Real-Time Log Tailing

`GET /api/logs/tail` returns the tail of a node's log. The Admin UI's per-node log viewer polls it every
two seconds, passing back the returned `offset` to read only what was appended since:

| Parameter | Meaning |
|---|---|
| `nodeType` | `master` (default) — the answering Master's own `.out` file — or `worker`. |
| `targetHost`, `targetPort` | For `nodeType=worker`: the Worker's advertised host and internal port, as `GET /api/nodes/workers` lists them. The Master fetches the log from the Worker over the internal NIO plane. |
| `lines` | Lines to return (default 200). |
| `offset` | Byte offset to continue from (default 0). |

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://mium:8090/api/logs/tail?nodeType=worker&targetHost=worker-1.internal&targetPort=19098&lines=500"
```

This means an SRE chasing an LLM error does **not** need SSH on the Worker host — every log surface is reachable through the Admin API.

## Logging Configuration

Mium uses Logback, configured by `conf/logback.xml`. The shipped config writes to **stdout only** — the launcher script (`start-master.sh` / `start-worker.sh`) redirects that to `logs/<service>[-<port>].out`. This keeps the container-native pattern consistent across `docker run`, systemd, and Kubernetes deployments.

To raise log levels at runtime without a restart, edit `conf/logback.xml` — Logback's scan reload picks the change up within 30 seconds.

## What to Alert On

| Signal | Source | Why |
|---|---|---|
| `/ready` returns `503` for > 1 min | LB probe | Cluster lost a node and is rejecting requests. |
| Heap used / max > 85% on any node | `/api/monitoring/metrics` | Headroom is gone — bump `-Xmx` in `conf/jvm.conf`. |
| Worker count = 0 | `/api/nodes/workers` | The Master will return `503` for every chat / export — exports require a Worker. |
| Leader changes more than once an hour | `/api/leader` | ZK or network instability — investigate before it becomes a quorum problem. |
| Any ZooKeeper server `DOWN` | `/api/nodes/zookeeper` | The ensemble has lost redundancy. With two of three down the survivor stops serving and Mium can no longer elect a leader. |
| ZooKeeper `latencyAvg` rising, or `outstanding` above 0 for long | `/api/nodes/zookeeper` | The ensemble is overloaded or its disk is slow — leader election and readiness will lag. |
| Many `EXECUTE_EXPORT` errors in `worker.out` | `logs/worker.out` | Python wheel ABI mismatch, missing `python3`, or S3 credentials wrong. |

## Browser Footprint

The Dashboard polls `/api/monitoring/metrics` once every few seconds while the page is open. The query is cheap — the in-memory time series is bounded, no database round-trip — but operators leaving the page open in many tabs see a real (small) load on the Master serving it. Close tabs you are not using.
