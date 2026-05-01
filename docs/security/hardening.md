# Security Hardening

Mium ships with sensible defaults for a single-host development install. Production deployments need to take a few additional steps to close the gap. This page walks through them in priority order — top of the list first.

## 1. Rotate the Bootstrap Admin

`mium.iam.admin.user = admin` / `mium.iam.admin.password = admin` exists so a fresh install can be logged into. **Rotate it immediately after first login.**

Two paths:

- Log in as `admin`, go to Settings → Profile → Password, set a strong password.
- Or via the API: `POST /auth/change-password` with the bootstrap credentials.

The bootstrap user is recreated only when the IAM RocksDB is empty, so changing the property after first boot has **no effect** — the password set at first login is the password that stands.

## 2. Protect the Master Key

`MIUM_MASTER_KEY` is the secret that derives the cluster's KMS root key. Treat it like a database master password.

- **Store it in your secret manager** (Vault, AWS Secrets Manager, GCP Secret Manager, …) — not in `.env`, not in `conf/mium.properties`, not in the Docker image.
- **Inject it at process start** — systemd `EnvironmentFile=`, Docker `--env-file`, Kubernetes `Secret` mounted as env.
- **Use the same value on every node**. Followers cannot decrypt the leader's KMS bundle without it.
- **Back it up separately from the data**. Losing the master key makes every encrypted store — IAM, KMS, ConnectionStore, MemoryStore, exports — unrecoverable.
- **Never commit it to source control.** Audit `git log` for accidental leaks before going to production.

Generate a strong key once:

```bash
openssl rand -base64 48 | head -c 32
```

The variable name is configurable via `mium.kms.master.key.env` — set it to whatever name your secret manager uses.

## 3. Terminate TLS at the Reverse Proxy

The browser ↔ Master HTTP path is plain HTTP. Mium does **not** terminate TLS itself. Front it with nginx, Envoy, an AWS ALB, or whatever your environment uses, and:

- Terminate TLS at the proxy.
- Forward to `mium.master.admin.port` (default `8090`) over the internal network.
- Set `X-Forwarded-For` and `X-Forwarded-Proto` so audit logs record the real client IP.

Internal NIO traffic (Master ↔ Master, Master ↔ Worker) is **already envelope-encrypted** under the cluster's KMS key once a key is available — no separate TLS layer is required for the control plane.

## 4. Lock Down Network Surface

| Port | Who Should Reach It |
|---|---|
| `8090` (Master HTTP) | Reverse proxy / load balancer only — **never** expose to the public internet. |
| `19099` (Master NIO) | Other Masters and Workers in the same cluster. |
| `19098` (Worker NIO) | Masters in the same cluster. |
| `2181` (ZooKeeper) | Mium nodes only — not a public surface. |

Workers and Masters typically run on a private subnet. The only port that ever needs to leave that subnet is `8090`, and only via the reverse proxy.

## 5. Use Least-Privilege IAM Policies

The shipped action vocabulary is granular enough to express tight policies. A few templates:

**Read-only end user.**

```json
{
  "statements": [
    {"effect":"ALLOW","actions":["MIUM:CHAT","MIUM:USE_CONNECTION","MIUM:READ_MEMORY","MIUM:READ_PROMPT"],"resources":["*"]},
    {"effect":"DENY","actions":["MIUM:MANAGE_*"],"resources":["*"]}
  ]
}
```

**Power user with prompt-library write.**

```json
{
  "statements": [
    {"effect":"ALLOW","actions":["MIUM:CHAT","MIUM:USE_CONNECTION","MIUM:USE_TOOL","MIUM:READ_MEMORY","MIUM:WRITE_MEMORY","MIUM:READ_PROMPT","MIUM:WRITE_PROMPT","MIUM:WRITE_CONNECTION"],"resources":["*"]}
  ]
}
```

**Cluster admin (full).** Attach to the bootstrap admin's group only — keep it small.

```json
{
  "statements": [{"effect":"ALLOW","actions":["MIUM:*"],"resources":["*"]}]
}
```

Explicit `DENY` always overrides `ALLOW`. Missing `ALLOW` means the request is rejected. There is no implicit-allow path.

## 6. Rotate KMS Keys on a Schedule

Mium's KMS supports **versioned keys** — rotation produces a new active version while older versions stay available for decrypting historical data. Schedule rotation on a cadence appropriate to your compliance regime (yearly for most, quarterly for stricter):

- Settings → Administration → Security & KMS → select key → Rotate.
- Or via API: `POST /api/kms/rotate`.

The rotation is online — no downtime, no re-encryption of existing data. New writes use the new version; reads use whichever version was active at write time.

## 7. Cap JWT Lifetime

`mium.jwt.ttl.seconds` defaults to `86400` (24 hours). For internal-only deployments that's fine. For public-facing or multi-tenant deployments shorten it:

```properties
mium.jwt.ttl.seconds = 3600     # 1 hour
```

Browsers will auto-refresh via `/auth/refresh` before expiry, so users don't notice. Stolen JWTs become useless faster.

## 8. Pin Connection Credentials to Per-User Connections

Mium's data access model is **never** shared credentials. Resist the temptation to install a single "team Ontul connection" under one user and have everyone reuse it — that loses the audit trail (Ontul sees one principal regardless of who actually queried) and concentrates blast radius.

Instead:

- Each user registers their own Ontul access key in their ConnectionStore.
- Ontul itself decides what data they can read.
- Mium does not federate IAM with Ontul, so authorization stays where it belongs — on the data engine.

## 9. Disable Retention Sweeps in Regulated Environments

Mium's retention sweep (`mium.memory.ttl.days`, etc.) is convenient but lossy. In environments where chat history is subject to legal hold or compliance retention, disable Mium's sweep entirely (set every `*.ttl.days` to `0`) and let your platform's own retention machinery — NeorunBase backups, S3 lifecycle, audit trails — own the lifecycle.

See [Retention & Sweeps](../operations/retention.md) for the full property list.

## 10. Audit the Connection Surface

`/api/connections` lists the calling user's connections. Operators can extend the policy to see other users' connections — and should periodically:

- Check for connections to unexpected hosts.
- Confirm every connection has a non-empty `tool` and a known `authType`.
- Revoke connections owned by deactivated users — Mium does not auto-revoke on user delete.

The Audit log in NeorunBase records every `POST /api/connections` and `DELETE /api/connections` action with a timestamp and principal.

## What Mium Will Not Do for You

- **Mium does not patch its own dependencies.** Watch for CVEs in your Java runtime, ZooKeeper, NeorunBase / PostgreSQL, and the bundled Python wheels. The distribution is reproducible — rebuild and redeploy on a security cadence.
- **Mium does not enforce TLS.** It's the operator's job to put a reverse proxy in front.
- **Mium does not enforce password complexity.** Pair the IAM with a corporate identity provider or rotate passwords on a schedule.
- **Mium does not auto-revoke access keys on user delete.** Run a periodic audit job to find dangling keys.
