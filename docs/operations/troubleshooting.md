# Troubleshooting

The five operational issues that account for most Mium support tickets, with the diagnostic that resolves each one in seconds.

## `/ready` Returns 503 — "Cluster Not Ready"

The single most common operational issue. `/health` is `200`, the JVM is up, but the load balancer is still rejecting traffic.

**Cause.** The leader Master accepts requests only when **every** registered Master and Worker has synced KMS / IAM / ConnectionStore from the leader and reported `ready=true` in ZooKeeper. Any node that registered but never reached ready holds the cluster offline.

**Diagnose.**

```bash
# Hit the leader directly
curl -H "Authorization: Bearer $TOKEN" http://leader:8090/api/nodes/masters
curl -H "Authorization: Bearer $TOKEN" http://leader:8090/api/nodes/workers
```

Each entry has a `ready` flag. If any flag is false, that's the holdout.

For the holdout node, check its log for one of:

- `MIUM_MASTER_KEY mismatch` — the master key on this node does not derive the same KEK as the leader. Fix: align the env var across all nodes.
- `Failed to pull KMS_SYNC` — network blocked between this node and the leader on `mium.master.internal.port` (default `19099`). Fix: open the firewall.
- `Schema migration failed` — NeorunBase is reachable but the user lacks DDL privileges. Fix: grant the `mium.neorunbase.username` permission to create tables in `mium.neorunbase.schema`.

**Recover.** Once the underlying cause is fixed, restart the holdout. The leader re-validates cluster readiness automatically; no leader restart is needed.

## Chat Returns 503 — "No Worker Ready"

The chat endpoint accepts the request but returns `503 Service Unavailable` immediately.

**Cause.** Every Worker is either unregistered, marked unready in ZooKeeper, or unreachable on its NIO port. The Master will not run LLM calls, tool calls, or exports locally — that work belongs on the Worker tier.

**Diagnose.**

```bash
curl -H "Authorization: Bearer $TOKEN" http://master:8090/api/nodes/workers
```

If the list is empty:

- A Worker has crashed and not restarted — check `bin/worker.pid` exists and the PID is still running.
- Workers were never started — `bin/start-worker.sh` was not run.
- ZooKeeper is unreachable from Worker hosts — workers cannot register without it.

If Workers are listed but `ready=false`, follow the same checks as the cluster-not-ready section above.

## Export Returns 503

The chat works, the LLM responds, but `Excel 로 만들어줘` returns `503` instead of a download link.

**Causes, in order of frequency.**

1. **No Worker ready** — same as above. Exports require a Worker.
2. **Python interpreter missing on Worker** — `python3` is not on PATH. Set `MIUM_PYTHON_EXE` to the absolute path.
3. **Bundled wheel ABI mismatch** — `lib/python/` was built against Python 3.11 but the Worker is running Python 3.10. Fix: install a matching `python3`, or override `MIUM_PYTHON_EXE`.
4. **S3 bucket unreachable** — the Worker can render but cannot upload. The Worker log shows the S3 error.
5. **S3 connection in `mium.tempfile.s3.connection` is missing or unauthorized** — admin needs to register the s3 connection with the right access key.

**Diagnose Python directly on the Worker host:**

```bash
$MIUM_HOME/bin/python3 -c "import openpyxl, reportlab, pptx; print('ok')"
```

## Login Fails — "Invalid Credentials"

The bootstrap admin works on a fresh install but fails after a config change.

**Cause.** The bootstrap admin (`mium.iam.admin.user` / `mium.iam.admin.password`) is created **only when the IAM RocksDB is empty**. Once any user has been seeded, changing those properties has no effect — the original credentials still apply. If the original password has been forgotten and not rotated through the UI, login is broken until you either:

- Reset the admin's password via another admin account.
- Restore the IAM RocksDB from backup.
- (Last resort, destructive) Stop the cluster, delete `data/iam/`, restart — re-seeds the bootstrap admin from `mium.properties`. This **wipes every user, group, policy, and access key**.

For non-bootstrap users, password reset is at Settings → Profile → Password (or `POST /auth/change-password`).

## Embedding Calls Hang or OOM on the Worker

Chat works but features that need embeddings (memory recall, prompt search, code-generation few-shot) stall or the Worker process gets killed.

**Cause.** Embedding inference loads `bge-m3` (~2 GB) and / or `clip-ViT-B-32` (~600 MB) into the Worker's Python heap. With Worker `-Xmx` plus model weights, total RSS approaches 8 GB. Hosts under-provisioned for that get killed by the OOM killer.

**Diagnose.**

```bash
# Are the daemons alive?
curl -H "Authorization: Bearer $TOKEN" http://master:8090/api/logs/tail?nodeId=worker-19098
# Look for: "embedding daemon ready: model=BAAI/bge-m3 dim=1024"
# Or:       "Killed" — that's the OOM killer.

# Check Worker memory headroom
ps -eo pid,rss,vsz,comm | grep python3
```

**Fix.** Either size up the Worker host, or use the lighter model:

```properties
# Settings → Embedding → switch to all-MiniLM-L6-v2 (~90 MB)
mium.embedding.dim = 384
```

The model swap is a hot swap — no restart needed. The first call after the swap pays the model-load cost (~5 s).

For the full embedding architecture, see [Worker Python Runtime](../features/worker-python-runtime.md).

## Other Useful Diagnostics

| Symptom | First check |
|---|---|
| Leader churns more than once an hour | `mium.zk.session.timeout.ms` and ZK ensemble health |
| Followers never catch up after a leader restart | NIO port `mium.master.internal.port` blocked between Masters |
| Export download is `404` | The 10-day retention sweep already deleted the file. Re-render. |
| `Schema "mium" does not exist` | NeorunBase connection has wrong schema name in `mium.neorunbase.schema` |
| Logs filling disk | Logback rotates daily; check `conf/logback.xml` for retention; mount `logs/` on a sized volume |
| Admin UI shows `Cannot GET /` | `mium.admin.context.path` set but the UI bundle was built without a matching `VITE_BASE_PATH` |

## Where to Look in Logs

| Subsystem | Tag in log line |
|---|---|
| Leader election | `LeaderLatch`, `Curator` |
| Master ↔ Master sync | `KmsSync`, `IamSync`, `ConnectionSync` |
| Master ↔ Worker dispatch | `WorkerToolDispatcher`, `WorkerEmbedDispatcher`, `ExecuteAgentHandler` |
| Agent loop | `AgentLoop`, `LlmBackend` |
| Export render | `PythonExporter`, `PythonProcessRunner` |
| Embedding daemon | `EmbeddingDaemon`, `EmbeddingTool` |
| HTTP request | `AdminHttpServer`, `<Endpoint>` |

Every Worker also reports its own logs back through `GET /api/logs/tail` — operators do not need SSH on the Worker host to chase a problem.
