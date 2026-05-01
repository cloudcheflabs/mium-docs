# Run Mium

Mium runs as two services on each host: **Master** and **Worker**. The Master serves the Admin UI and the REST API, owns IAM / KMS / ConnectionStore on RocksDB, and coordinates Workers. Workers execute LLM calls, tool calls, and server-side export rendering.

This page walks through a single-host start. For multi-host clustering see [HA Cluster](ha-cluster.md).

## Prerequisite: Running ZooKeeper

Mium will not boot without a ZooKeeper ensemble. In production point `mium.zk.connect` at your existing ZK ensemble. For development and smoke tests the package ships an embedded ZK helper script:

```bash
bin/start-zk.sh        # uses conf/zk/zoo.cfg, data under ./data/zookeeper
```

```bash
bin/stop-zk.sh
```

## Start the Master

```bash
bin/start-master.sh
```

The script:

- Reads JVM options from `conf/jvm.conf`.
- Sets `MIUM_HOME`, `mium.config.file`, and `mium.log.path` automatically.
- Writes a PID file to `bin/master[-PORT].pid` (the port suffix lets multiple instances coexist on the same host).
- Redirects stdout/stderr to `logs/master[-PORT].out`.

Watch the log until you see `Mium Master is ready`:

```bash
tail -f logs/master.out
```

## Start the Worker

```bash
bin/start-worker.sh
```

PID file: `bin/worker[-PORT].pid`. Log: `logs/worker[-PORT].out`. The Worker registers itself with ZooKeeper and pulls KMS, IAM, and ConnectionStore from the leader Master before reporting `ready=true`.

The leader Master will not accept user requests until **every** registered Worker (and every follower Master) reports ready — see [Troubleshooting](../operations/troubleshooting.md).

## Verify Health

```bash
curl http://localhost:8090/health
# {"status":"ok","service":"mium-master"}

curl http://localhost:8090/ready
# 200 once the cluster is ready, 503 until then
```

`/health` is a liveness probe and always returns 200 once the JVM is up. `/ready` returns 503 until KMS / IAM / ConnectionStore have synced and every node is ready, and is the right probe for load balancers.

## Log In

Open `http://<master-host>:8090/` in a browser. Log in with the bootstrap admin from `mium.properties`:

```
mium.iam.admin.user     = admin
mium.iam.admin.password = admin
```

**Rotate the bootstrap password immediately** via Settings → Profile → Password (or `POST /auth/change-password`). The bootstrap user is recreated only when the IAM RocksDB is empty, so a forgotten password is a destructive recovery — see [Backup & Restore](../operations/backup-restore.md).

## Stop the Services

```bash
bin/stop-worker.sh
bin/stop-master.sh
```

Both scripts SIGTERM the process, wait up to 30 seconds for graceful shutdown, then SIGKILL if needed. The PID file is removed on success.

## Run in the Foreground

For systemd / Docker / Kubernetes the launchers can run inline instead of forking:

```bash
MIUM_FOREGROUND=true bin/start-master.sh
MIUM_FOREGROUND=true bin/start-worker.sh
```

In foreground mode the script `exec`s the JVM, so it inherits the parent process's stdin/stdout and PID. No `.pid` file is written.

## Multiple Instances on One Host

The launcher scripts derive a port suffix for the PID file from `-Dmium.master.admin.port=...` or `-Dmium.worker.internal.port=...`, so multiple Masters or Workers can coexist on the same host:

```bash
bin/start-worker.sh -Dmium.worker.internal.port=19098      # → bin/worker-19098.pid
bin/start-worker.sh -Dmium.worker.internal.port=19198      # → bin/worker-19198.pid
```

The matching `bin/stop-worker.sh 19198` stops only the second instance.

## Quick Smoke Test

Once you're logged in:

1. Settings → LLM Backends → register an Anthropic or Ollama connection.
2. Settings → Tool Connections → register your Ontul connection (Arrow Flight access key).
3. Open the Chat page and ask a SQL question — e.g. *"List the top 10 rows of `tpch.tiny.orders`."*

If the LLM emits a `query` action and Mium returns rows, the install is working end-to-end. If the chat returns `503 Service Unavailable`, no Worker is ready — check `bin/worker.pid` and `logs/worker.out`.
