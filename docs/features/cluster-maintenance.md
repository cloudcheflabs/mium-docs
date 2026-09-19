# Cluster Maintenance Mode

A cluster-wide switch that stops Mium **writing**, so an operator can restore a backup, rotate keys, or replace workers without a write landing in the middle of the work.

It does not close the cluster. Reads keep being served, settings stay editable, running jobs are not killed, and backups keep going.

## What it refuses

| Refused with `503` | Left open |
| --- | --- |
| `POST /api/chat` — an agent turn | Reading chat history and session lists |
| `POST /api/chat/sessions` — create | Catalog, semantic, monitoring, node topology |
| `POST /api/chat/sessions/rename` | `POST /api/jobs/kill`, `/status`, `/logs`, `/list` |
| `DELETE /api/chat/sessions` and `/sessions/all` | IAM, KMS, connections, embedding settings |
| `POST /api/jobs/submit`, `/codegen`, `/codegen-fix` | Backup configuration and manual runs |

The rule in one line: **writes are refused; reads and settings are not.**

An agent turn is the heaviest write Mium makes, which is why it heads the list. It persists chat memory, and through tools it reaches out to Ontul, GitHub, Slack and whatever else the user has connected — so a turn running during a restore can write to systems well outside Mium.

**Job kill stays open,** as it does in every product here: the first thing an operator does after opening a window is stop the work still running. A switch that blocked it would be blocking the work it exists to enable.

## The retention sweep pauses

This is the part that would be easy to leave out and expensive to get wrong.

Mium runs a leader-only retention sweep that deletes expired chat memory, prompts and temp files by TTL. It is the one background loop in Mium that **destroys** data — and during a restore it would delete, by TTL, exactly the memory and prompts the restore had just brought back. Worse, the operator would have no way to tell that from the restore having failed.

So the sweep checks the window on every tick and skips while it is open:

```
INFO  MasterServer - Skipping retention sweep — the cluster is in maintenance mode
```

It resumes on the next tick after the window closes. Nothing is deferred and nothing accumulates: the sweep is idempotent by design — it deletes whatever is past its TTL whenever it next runs.

Everything else in the background keeps going. Metrics collection is observation, backups are usually the reason the window was opened, and the follower snapshot pull has to keep running or the flag itself would not propagate.

## Where the setting lives

In the `mium_settings` table — the same generic `key`/`value`/`version` row that holds the embedding settings, which exists precisely so small admin settings can share it. NeorunBase is a hard requirement for Mium, so the row is always reachable.

!!! note "Why not RocksDB, when the other products use it"
    ShannonStore, Ontul and Kiok store this flag in their RocksDB config store. Mium's RocksDB is not the same kind of thing: it holds KMS, IAM and connections, each in its own store, with no general config store to put a single flag in. Mium's general settings store is this table.

    That choice also buys latency. Follower masters pull the RocksDB snapshot on a 30-second timer, so a flag kept there would take up to half a minute to reach them. This row is visible to every master **and worker** within a one-second cache, with no replication machinery at all.

    What does not change is the part that matters, and it is the same rule across every Cloud Chef Labs product: **ZooKeeper holds node state** — membership, leadership, readiness — and **settings live in a durable store of ours.** Only the identity of "our durable store" differs here.

Being stored rather than held in memory is what makes it survive a master restart, a full cluster restart, and a master joining mid-window.

### It fails open, not closed

If the settings row cannot be read — the database is briefly unreachable, the table is missing on first boot — the node treats the cluster as **open** and logs a warning.

A switch that failed closed would turn a momentary database blip into a cluster-wide outage, which is a far worse failure than briefly accepting a write during a maintenance window.

## Turning it on and off

From the Admin UI: **Nodes → Enter Maintenance**. A confirmation appears first, and a banner stays on screen while the window is open.

Or over REST:

```bash
TOKEN=$(curl -sf -X POST http://localhost:8090/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"admin","password":"…"}' | jq -r .accessToken)

# status
curl -sf http://localhost:8090/api/admin/maintenance -H "Authorization: Bearer $TOKEN"

# on
curl -sf -X POST http://localhost:8090/api/admin/maintenance \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":true}'

# off
curl -sf -X POST http://localhost:8090/api/admin/maintenance \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":false}'
```

`POST` requires `SYSTEM:MANAGE_MAINTENANCE` on `system:maintenance` and is routed to the leader, which keeps the settings row single-writer.

`GET` is answered by **whichever master receives it**, from that master's own cache — deliberately not proxied to the leader. An operator opening a window wants to confirm every master agrees, and a status route that quietly asked the leader could only ever say yes.

## What clients see

```
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{"error":"cluster is in maintenance mode; writes are refused",
 "maintenanceMode":true,"retryAfterSeconds":30}
```

`503` with `Retry-After` rather than `403`: the caller is not forbidden, the cluster is closed for now, and every HTTP client already knows to back off on that pair.

```properties
# How long a client is told to wait before retrying a refused write (seconds).
mium.cluster.maintenance.retry.after.seconds = 30
```

## When to use it

| Use it for | Don't use it for |
| --- | --- |
| Restoring a backup over live chat memory and IAM state | Adding a master or worker |
| KMS rotation | A rolling upgrade |
| Replacing workers as a group | A single worker restart |
| Investigating a problem without new turns piling in | Anything shorter than a client's retry budget |

## Verifying it

`tests/maintenance-mode-e2e.sh` in the product repository runs the cycle against a compose stack of Postgres, ZooKeeper and one master: a chat-session write **succeeds first**, the window opens and the same write is refused with `503` + `Retry-After` along with the agent turn, rename, delete and job submission, the retention sweep logs that it is skipping, reads and job kill and the status route keep working, the master is restarted and comes back still refusing, and then the window closes and the write succeeds again while the sweep resumes.

The test does **not** assert what `POST /api/chat` returns with the window open, because a real agent turn needs a configured LLM and that stack deliberately has none. That gap is stated in the test rather than hidden: all five gates are the same one-line check, and the chat-session probe exercises that check's full lifecycle with a genuine success on both sides of the window.

## See also

- [Backup &amp; Restore](backup-restore.md) — the operation a window is most often opened for.
- [Chat Memory](chat-memory.md) — what the retention sweep deletes.
- [High Availability](high-availability.md) — leadership and what is replicated between masters.
