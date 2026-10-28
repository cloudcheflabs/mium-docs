# Chat & Memory

Mium provides a persistent chat system where users interact with LLMs through a conversational interface. Chat history is stored in the MemoryStore, enabling context-aware conversations across sessions.

## Chat Interface

The Admin UI provides a full-featured chat interface at the root path:

- Create new chat sessions
- Continue previous conversations with full context
- Delete and rename sessions
- Export of the result payload: **CSV and Markdown are produced client-side** (inline preview / browser download); **XLSX, PDF, and PPTX are rendered server-side** on a Worker and returned as a `downloadUrl` (see [Server-Side Export](server-side-export.md))

## Chat Sessions

Each chat session maintains:

- **Message history** — user and assistant turns in order.
- **Tool invocations** — records of tool calls and their results, surfaced as intermediate assistant messages so users can see the full reasoning chain, not just the final answer.
- **Rolling context** — the agent loop sends the LLM the most recent turns bounded by a fixed `MAX_HISTORY = 20` window (not configurable today).
- **Workspace** — each session belongs to a workspace (`chat`, `analyze`, or `dev`); intent auto-routing can move a turn into a new session in a different workspace.

Sessions are identified by unique IDs and can be resumed at any time.

## Semantic Recall

When an embedding backend is available, Mium embeds each turn and, on later turns, retrieves the top-K (default 5) most relevant past excerpts from the user's `memory:` namespace and injects them into the prompt — giving the agent cross-session recall beyond the fixed rolling window. This is gated on the embedding subsystem being ready; without it, only the rolling window is used.

## MemoryStore

The MemoryStore persists chat data in **NeorunBase** (`mium_chat_session` and `mium_chat_message` tables). Every Mium node reads and writes directly — no Mium-side replication needed because NeorunBase handles durability natively. See [Storage Backends](storage-backends.md) for the full store layout.

## Chat API

The Chat endpoints on the Admin HTTP server (mounted under `/api/chat` by default; a `/admin` prefix applies only when `mium.admin.context.path` is set) provide:

- `POST /api/chat` — send a message and receive a reply (with optional tool invocations / `downloadUrl`)
- `GET /api/chat/sessions` — list the current user's sessions (`/api/chat/sessions/all` for the full set)
- `GET /api/chat/sessions/messages` — get a session's full message history
- `POST /api/chat/sessions/rename` — rename a session; delete via the sessions endpoint

Writes are leader-only; followers transparently proxy via `LeaderRouter`.

## Secrets in Chat

Registering a catalog or a connection through chat hands a password, an S3 secret key or a token to a
tool. Mium keeps that secret out of everything it shows or stores afterwards:

- **Replies and payloads** — a catalog config is shown with every credential value as `***`
  (`"password" : "***"`), and so are `list_catalogs`, the `ontul_admin` / `mium_admin` responses
  (for example a new Ontul access key's `secretAccessKey` and `token`) and connection endpoints
  (`...?password=***`). The keys that count as credentials are Ontul's: anything containing
  `password`, `secret`, `token`, `credential`, `apikey`, `accesskey`, `private` or `jaas`.
  If the model repeats a secret in its own explanation, that is replaced with `***` too.
- **Sending `***` back** — `update_catalog` can send a secret as `***`, and Ontul keeps the stored
  value. Changing a catalog's URL therefore never requires typing its password again.
- **The model** — after a secret has been sent to a tool, the rest of the turn's model calls and every
  later turn are sent without it. Only the turn that collects it (and the confirmation before
  registering) carries it.
- **Chat history** — the conversation is rewritten with `***` in place of the secret, including the
  user's own message that typed it ("the password is ***"), and the semantic-recall copy of a
  rewritten message is dropped. Exports, which render query results, never contain a tool config.
- **Logs** — Mium does not log tool configs; a model reply quoted in a warning is masked first.

History written by a release older than this is swept **once**, the first time a leader starts on the
new release: credential values in stored catalog/admin replies are masked, and every message of the
same conversation that repeats one of those values — the user's own turn included — is rewritten.
`dev` workspace notebooks are left untouched (a cell is the user's code). The sweep is recorded in
`mium_settings` (`chat-secret-scrub-v1`); deleting that row and restarting the leader runs it again.

!!! note "What the masking cannot cover"
    A secret typed into chat reaches the LLM provider in that turn: the model has to read it to build
    the catalog config. To keep a secret away from the model entirely, register the catalog in Ontul's
    console or create a Mium connection under **Settings → Tool Connections** and refer to it by name.

## Retention and Compaction

Long-running chats are kept manageable by the leader's retention sweep. The sweep is **paused while
the cluster is in [maintenance mode](cluster-maintenance.md)** — it is the one background loop that
deletes data, and during a restore it would remove exactly what the restore just recovered:

- **TTL sweep** — sessions older than `mium.memory.ttl.days` are dropped (`0` = disabled).
- **Per-user session cap** — each user's oldest sessions are evicted beyond `mium.memory.user.maxSessions` (`0` = unlimited).
- **Per-session turn cap** — a session over `mium.memory.session.maxTurns` (default 200) has its oldest turns truncated.

!!! note "Compaction is not yet wired"
    `MemoryCompactor` / `Summarizer` (LLM summarisation of older turns) and the `mium.memory.compaction.*` settings exist in the codebase but are **not** invoked by the running retention sweep today — treat LLM-driven compaction as dormant/planned. The active retention mechanisms are the TTL sweep, the session cap, and the per-session turn cap above.
