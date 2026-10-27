# Admin Password Recovery

mium ships with a built-in recovery channel that lets an operator reset the
`admin` user's password **without stopping the master**, even when no one
remembers the current password.

Recovery uses a local Unix domain socket — there is no HTTP back-door, no
network endpoint, no recovery URL. Authentication is performed by the
operating system: only a process that already shares the master's filesystem
identity can open the socket.

## When to use

- The admin password was forgotten or rotated out of the password manager.
- An automation script needs to provision a known admin password during
  first-time setup, without going through the web UI.
- A new operator is onboarded and you need to hand them an admin credential.

For day-to-day password changes (the user remembers the old password and
wants a new one), use the Admin UI's **Change Password** screen instead —
that path requires the old password and does not flag the user for forced
rotation.

## How it works

```
┌────────────────────┐   JSON over UDS    ┌────────────────────┐
│  mium-cli          │ ─────────────────▶ │  Master (running)  │
│  iam:reset-password│                    │   AuthManager      │
└────────────────────┘                    │ .adminResetPassword│
         ▲                                │                    │
         │ stdout: new password           │  saveToDb()        │
         │ (one-time)                     │  cluster sync push │
                                          │  audit log append  │
                                          └────────────────────┘
```

| Property | Value |
|---|---|
| **Socket path** | `${mium.base.data.dir}/admin.sock` by default, mode `600` — the live value is published to `bin/master.socket` |
| **Authentication** | OS file permission — same user as the master process |
| **Socket path marker** | `bin/master.socket` — written when the socket binds, removed on shutdown |
| **Network surface** | none — Unix domain socket only |
| **Downtime** | none — applied in-process on the live master |
| **Cluster sync** | IAM is leader-owned: on a follower the reset is forwarded to the leader, which applies it and fans it out to every master and worker; with no reachable leader the CLI fails (exit 1) and nothing changes |
| **Audit log** | `data/iam-audit/reset.log` (mode `600`, append-only) |
| **Post-reset state** | `requirePasswordChange = true` (forced rotation on next login) |

## Quick start

The simplest invocation lets the master generate a strong 20-character
password and print it to stdout. The new password must be changed on the
admin's next login (the `requirePasswordChange` flag is set automatically).

```bash
# Inside the master host or container:
bin/mium-cli.sh iam:reset-password
```

stdout carries only the generated password, so a script can capture it. On
stderr the CLI says what the account must do before the API answers it:

```text
note: 'admin' must change this password once before using the API: log in with it
(POST /auth/login {"username":"admin","password":"<this password>"}), then POST /auth/change-password
with header 'Authorization: Bearer <accessToken>' and {"oldPassword":"<this password>","newPassword":"<new>"}
(paths are under mium.admin.context.path when one is set). Until then every other endpoint answers
403 "password change required". The console asks for it at the first login.
```

For a script:

```bash
PW=$(bin/mium-cli.sh iam:reset-password 2>/dev/null)
TOKEN=$(curl -s -X POST http://<master>:8090/auth/login -H 'Content-Type: application/json' \
  -d "{\"username\":\"admin\",\"password\":\"$PW\"}" | jq -r .accessToken)
curl -s -X POST http://<master>:8090/auth/change-password -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d "{\"oldPassword\":\"$PW\",\"newPassword\":\"<new>\"}"
```

### Which java the CLI and the start scripts run

`bin/mium-cli.sh`, `bin/start-master.sh`, `bin/start-worker.sh` and
`bin/start-zk.sh` run `$JAVA_HOME/bin/java` when `JAVA_HOME` is set, and the
`java` on `PATH` otherwise — a host with the JDK only in `JAVA_HOME` works.
`JAVA_HOME` pointing at a directory without `bin/java` stops them with exit 1
naming the value; no `JAVA_HOME` and no `java` on `PATH` stops them with
*Java not found. Install Java 17 or later, or set JAVA_HOME to its directory.*

## Input modes

| Mode | Command | When to use |
|---|---|---|
| **Master-generated** | `mium-cli.sh iam:reset-password` | Default. Strong random password printed once on stdout. |
| **Explicit** | `mium-cli.sh iam:reset-password --new-password 'My!Pass'` | Automation that knows the desired value. Beware: argv may show up in `ps`. |
| **Stdin** | `echo 'My!Pass' \| mium-cli.sh iam:reset-password --new-password -` | Automation that wants to avoid argv exposure. |
| **Interactive** | `mium-cli.sh iam:reset-password --interactive` | Operator at a TTY. Prompts for password twice with no echo. |

Resetting a different user is also supported:

```bash
bin/mium-cli.sh iam:reset-password --user some-user --new-password 'NewPass123'
```

## Configuration

The recovery socket is enabled by default. Every key below lives in
`conf/mium.properties` and is read at master startup:

```properties
# conf/mium.properties
# Set false to remove the local recovery path entirely.
mium.admin.socket.enabled      = true
mium.admin.socket.path         = ${mium.base.data.dir}/admin.sock
# Name of the file under <mium.home>/bin that receives the socket path the
# master actually bound to (see "How the CLI finds the socket" below).
mium.admin.socket.marker.file  = master.socket
# How long a follower waits for the leader to apply a forwarded reset
# (see "Running it on a follower master" below).
mium.admin.socket.forward.timeout.ms = 30000
# Append-only audit trail of socket operations.
mium.iam.audit.dir = ${mium.base.data.dir}/iam-audit
```

The socket follows `mium.base.data.dir`. If you launch the master with
`-Dmium.base.data.dir=/var/lib/mium` the socket moves to
`/var/lib/mium/admin.sock` — you do not have to restate it.

### Running it on a follower master

You can run the CLI on any master. IAM state is owned by the leader, so a
**follower forwards the reset to the leader** over the internal protocol; the
leader applies it and pushes it to every master, and the follower checks that
its own copy now holds the new password before answering. The CLI says where
it was applied:

```text
$ bin/mium-cli.sh iam:reset-password --new-password 'FromFollower1!'
note: this master is a follower; the reset was applied on the leader mium-master-1:8090 (mium-master-1:19099) and replicated to the cluster.
```

`bin/mium-cli.sh ping` tells you which role the local master has:

```text
pong (follower mium-master-2:8090; leader: mium-master-1:8090 (mium-master-1:19099))
pong (leader mium-master-1:8090)
```

When no leader is elected, or the leader does not answer within
`mium.admin.socket.forward.timeout.ms` (default 30 s), the CLI **exits 1**,
names the leader and states that the password was **NOT** changed:

```text
error: this master (mium-master-2:8090) is a follower and the leader mium-master-1:8090 (mium-master-1:19099)
did not apply the reset (no answer within 30000 ms); the password was NOT changed. ...
```

A forwarded reset carries a deadline of one third of that wait. A leader that
receives it later — it was paused, overloaded or partitioned — discards it, so a
reset the CLI already reported as failed never lands afterwards. Keep the
masters' clocks in sync (NTP). Run the CLI again once a leader is up, or run it
on the leader host.

!!! note "Before this change"
    A reset run on a follower changed only that follower's copy and exited 0; the
    follower's next snapshot pull from the leader (within about 30 seconds) put the
    old password back.

### How the CLI finds the socket

Re-deriving the socket path from `conf/mium.properties` is not reliable on its own:
`mium.base.data.dir` can be overridden with `-D` at launch or edited after
startup, and the file does not record which value the live process used. So the
master **publishes the path it actually bound to** into
`<install dir>/bin/master.socket` when the socket comes up, and removes that file on
shutdown. `bin/mium-cli.sh` prefers it.

Full resolution order, highest priority first:

1. `--socket /path/to/admin.sock` — read by the Java CLI, always wins.
2. `$MIUM_ADMIN_SOCKET` — if already exported in the caller's shell.
3. `<install dir>/bin/master.socket` — the path published by the running master.
   Used only when the file exists *and* the path in it is a live socket.
4. `mium.admin.socket.path` from `conf/mium.properties`, with
   `${mium.base.data.dir}` expanded. A value that still contains a
   `${...}` placeholder is rejected rather than used literally.
5. `<install dir>/data/admin.sock`, then `/data/admin.sock`.

Step 3 is what makes a moved data dir work: with the socket at
`/data/admin.sock` and the properties file still saying `./data`, only the marker
knows where to connect.

To rename the marker, change one key — both ends read it:

```bash
# conf/mium.properties
mium.admin.socket.marker.file = mium-recovery.socket
```

Restart the master; it publishes `bin/mium-recovery.socket`, and the CLI
picks the new name up from the same properties file.

### The master key is for the master, not the CLI

`MIUM_MASTER_KEY` must be exported for the master process. The start script does not pre-check it, but KMS is enabled by default and the master cannot unseal its keystore without the key, so startup fails during KMS initialisation.
The variable name itself is configurable — `mium.kms.master.key.env` in
`conf/mium.properties` names the variable the master reads:

```bash
export MIUM_MASTER_KEY='replace-with-a-32-char-or-longer-secret'
bin/start-master.sh
```

`bin/mium-cli.sh` does **not** need it. The CLI only opens the Unix socket and
hands the request to the running master, which already holds the unsealed key,
so this works with the variable unset:

```bash
unset MIUM_MASTER_KEY
bin/mium-cli.sh ping
# pong
```

If a CLI invocation complains about the key rather than the socket, you are
running a start script, not the CLI.

### Worked examples

```bash
# 1. On the host, as the same OS user that runs the master:
cd /opt/mium
bin/mium-cli.sh ping
bin/mium-cli.sh iam:reset-password

# 2. The master runs as a service account and you are root:
sudo -u mium /opt/mium/bin/mium-cli.sh iam:reset-password

# 3. Inside a container:
docker exec -it mium-master-1 /app/bin/mium-cli.sh iam:reset-password

# 4. Data dir was relocated at launch — no extra flags needed, the CLI
#    reads the published marker:
cat /opt/mium/bin/master.socket
# /var/lib/mium/admin.sock
bin/mium-cli.sh ping

# 5. Socket in a non-standard place and no marker (the master is stopped,
#    or you are on a host where the marker was cleaned up):
bin/mium-cli.sh --socket /var/lib/mium/admin.sock iam:reset-password

# 6. Non-interactive automation, password from stdin so it never reaches argv:
echo 'S0me!Strong!Pass' | bin/mium-cli.sh iam:reset-password --new-password -
```

## Security model

**1. The socket is OS-gated.**
At startup the master creates `data/admin.sock` with mode `600` (owner
read/write only). Even other unprivileged users on the same host cannot
connect. There is no token, no shared secret, no network listener.

**2. The audit log records every reset.**
Every successful reset appends a JSON line to `data/iam-audit/reset.log`
(mode `600`). The plaintext password is **never** logged, nor any part of
the stored hash — only a fingerprint (the first 12 hex characters of the
SHA-256 of the stored hash), the user, whether it was master-generated, the
OS user that invoked the CLI, and, in a cluster, the route: `forwardedTo=` on
the master where the CLI ran, `forwardedFrom=` on the leader that applied it.

```json
{"ts":"2026-10-27T10:57:02.114Z","event":"iam.reset-password","user":"admin","generated":false,"hashFp":"3f9c0a1b7d22","invokedAs":"root","route":"forwardedFrom=mium-master-2:8090"}
```

Two audit lines with the same `hashFp` refer to the same password version.

**3. The new password is exposed exactly once.**
For master-generated passwords, the plaintext is returned only on the
single CLI invocation that triggered the reset. It is not written to the
audit log, not stored in RocksDB, and not retransmitted. Treat scrollback
and shell history accordingly — or use stdin input mode to avoid argv
exposure entirely.

**4. Forced rotation on next login.**
After reset, the user is flagged `requirePasswordChange = true`. The next
successful login forces the user through the change-password flow, so a
temporary password used by the operator is immediately replaced by a
password only the user knows.

**5. The master must be running.**
Because the recovery channel is in-process, the master must be alive for
the CLI to connect. This is intentional: RocksDB requires an exclusive
lock, so an offline edit would either conflict with a running master or
need a complex stale-lock recovery. With this design, the only way to
reset is to be on the host *and* have the master running *and* share its
filesystem identity.

## Limitations

- **Master must be running.** If the master is down, this CLI cannot help.
  Bring the master back up first, then run the CLI.
- **No knowledge factor.** Any process that shares the master's filesystem
  identity can invoke the CLI. In multi-tenant or shared-shell
  environments, restrict shell access to the master accordingly. A future
  enhancement may add an opt-in "recovery key" requirement for an
  additional knowledge factor.

## Related

- [IAM](iam.md)
- [High Availability](high-availability.md)
- [Encryption](encryption.md)
