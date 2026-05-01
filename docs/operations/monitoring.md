# Monitoring & Observability

Mium reports cluster health through a single Master endpoint and surfaces it on the Admin UI Dashboard. There is no Prometheus exporter, no separate metrics agent — the Master collects metrics from every node over the internal NIO protocol and serves them on the same HTTP port the UI uses.

## What Is Collected

Every node (Master and Worker) reports a small set of JVM and process metrics on a fixed cadence:

| Metric | Source |
|---|---|
| Process CPU % | `OperatingSystemMXBean.getProcessCpuLoad()` |
| Heap used / committed / max | `MemoryMXBean.getHeapMemoryUsage()` |
| Non-heap used | `MemoryMXBean.getNonHeapMemoryUsage()` |
| Thread count | `ThreadMXBean.getThreadCount()` |
| Uptime | `RuntimeMXBean.getUptime()` |

Workers push their snapshots to the leader Master via the `METRICS_REQ` opcode. The leader keeps a rolling time-series in memory and serves it on `GET /api/monitoring/metrics` for the Dashboard's Recharts surface.

The metric format mirrors Ontul's so that operators familiar with the broader CCL stack see a consistent shape across products.

## Topology Probes

| Endpoint | Purpose |
|---|---|
| `GET /api/nodes/masters` | Live list of Master nodes with leader flag and ready state. |
| `GET /api/nodes/workers` | Live list of Worker nodes with ready state. |
| `GET /api/leader` | Current leader's node id and address. |

The Topology page in the Admin UI is built directly on these — anything you see there can be scripted.

## Health and Readiness Probes

| Endpoint | Purpose |
|---|---|
| `GET /health` | Liveness. Returns `200` once the JVM is up. Use for process-supervisor / Docker `HEALTHCHECK`. |
| `GET /ready` | Readiness. Returns `200` only when the cluster is whole — every Master and Worker has synced KMS / IAM / ConnectionStore. Returns `503` until then. Use for load-balancer health checks. |

A common mistake is to point the load balancer at `/health` — that traps traffic on a Master that is up but cannot serve writes (e.g. the leader is down). Always use `/ready` for LB health, `/health` for process supervisors.

## Real-Time Log Tailing

`GET /api/logs/tail?nodeId=<id>&lines=200` streams the tail of any node's `.out` file back to the caller. The Admin UI uses this for the per-node log viewer; operators can hit it from a terminal:

```bash
curl -sN -H "Authorization: Bearer $TOKEN" \
  "http://mium:8090/api/logs/tail?nodeId=worker-19098&lines=500"
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
| Many `EXECUTE_EXPORT` errors in `worker.out` | `logs/worker.out` | Python wheel ABI mismatch, missing `python3`, or S3 credentials wrong. |

## Browser Footprint

The Dashboard polls `/api/monitoring/metrics` once every few seconds while the page is open. The query is cheap — the in-memory time series is bounded, no database round-trip — but operators leaving the page open in many tabs see a real (small) load on the leader. Close tabs you are not using.
