# Job Lifecycle

In addition to ad-hoc SQL, Mium's agent loop can drive the full lifecycle of long-running Ontul jobs from a chat conversation. The LLM emits one of the job-action verbs in its strict-JSON reply and Mium dispatches the matching call against the user's Ontul connection.

## Action Vocabulary

| Action | What It Does |
|---|---|
| `submit_batch` | Submit a one-shot SQL job (CTAS, INSERT INTO, aggregations producing a target table). Returns `jobId` and initial `status`. |
| `submit_streaming` | Submit a continuous job that consumes from a streaming source. Returns `jobId` and initial `status`. |
| `job_status` | Look up the current state of a job by id. Returns `status` (`SUBMITTED`, `PLANNING`, `RUNNING`, `COMPLETED`, `FAILED`, `KILLED`), `durationMs`, `errorMessage` if applicable. The terminal success state is `COMPLETED`; `isTerminal()` covers `COMPLETED`, `FAILED`, `KILLED`. |
| `job_logs` | Stream log lines for a job, optionally from an offset. |
| `kill_job` | Terminate a running job by id. |
| `list_jobs` | Active jobs visible to the calling user. |
| `list_history` | Past jobs (terminal status). |
| `generate_code` | Produce Ontul SDK source for the user to copy or submit themselves. Supports `jobType` = `BATCH`, `STREAMING`, `CLASS`, `PYTHON`. |

## Conversational Pattern

Jobs are tracked across turns by `jobId`. Within a single chat session the LLM picks the id from earlier replies, so:

```
User:    "tpch.tiny.orders 를 orderstatus 별로 count 집계하는 BATCH job 을 이름 seg_count 로 제출해줘."
Agent:   submit_batch   → jobId: e5240f5d-…, status: SUBMITTED

User:    "방금 제출한 job 상태 확인해줘."
Agent:   job_status     → status: COMPLETED, durationMs: 301

User:    "job history 보여줘."
Agent:   list_history   → [{jobId: e5240f5d-…, status: COMPLETED, …}]
```

Across new sessions the user typically retrieves the id via `list_history` first.

## Code Generation

`generate_code` does NOT submit anything — it returns Ontul SDK source the user can copy, version, or hand to a different submit pipeline. It's the right action when the user asks "write me the Java" or "give me a Python script that …".

## REST Jobs API (Dev workspace)

Besides the chat-agent verbs above, the Admin UI's **Dev** workspace drives jobs through a dedicated REST surface on the master (default context path):

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/jobs/codegen` | Generate Ontul SDK source from a natural-language prompt |
| `POST` | `/api/jobs/codegen-fix` | Feed an error back to the generator for a fix-it iteration |
| `POST` | `/api/jobs/submit` | Submit a job in one of the `Mode` values below |
| `GET`  | `/api/jobs/status` | Poll job status |
| `GET`  | `/api/jobs/logs` | Fetch job logs |
| `GET`  | `/api/jobs/list` | List jobs |
| `POST` | `/api/jobs/kill` | Kill a job |

`POST /api/jobs/submit` accepts a `Mode`: `QUERY`, `BATCH`, `STREAMING`, `REMOTE_JAVA`, `REMOTE_PYTHON`, `CLUSTER_JAVA`, `CLUSTER_PYTHON`. The `REMOTE_*` modes run user-supplied Java/Python as a **Mium-side subprocess** via `RemoteJobRunner` (job ids prefixed `mium-remote-`), independent of Ontul cluster jobs; `CLUSTER_*` and `BATCH`/`STREAMING`/`QUERY` go to the Ontul cluster.

## Per-User Credentials

All job actions go through the user's Ontul connection in the ConnectionStore (`tool=ontul`). The Ontul cluster decides what the calling principal is allowed to do — Mium does not federate IAM with Ontul, so authorization comes from Ontul's own access-control layer.
