# Chat & Memory

Mium provides a persistent chat system where users interact with LLMs through a conversational interface. Chat history is stored in the MemoryStore, enabling context-aware conversations across sessions.

## Chat Interface

The Admin UI provides a full-featured chat interface at the root path:

- Create new chat sessions
- Continue previous conversations with full context
- Delete and rename sessions
- Client-side export of the result payload to CSV, Markdown, XLSX, PDF, or PPTX

## Chat Sessions

Each chat session maintains:

- **Message history** — user and assistant turns in order.
- **Tool invocations** — records of tool calls and their results, surfaced as intermediate assistant messages so users can see the full reasoning chain, not just the final answer.
- **Rolling context** — Mium trims the LLM context window with a configurable cap; long sessions can also be compacted by the `MemoryCompactor` which summarises older turns into a single synthetic message.

Sessions are identified by unique IDs and can be resumed at any time.

## MemoryStore

The MemoryStore persists chat data in **NeorunBase** (`mium_chat_session` and `mium_chat_message` tables). Every Mium node reads and writes directly — no Mium-side replication needed because NeorunBase handles durability natively. See [Storage Backends](storage-backends.md) for the full store layout.

## Chat API

The Chat endpoints on the Admin HTTP server (mounted under `/admin/api/chat`) provide:

- Send a message and receive a reply (with optional tool invocations)
- List sessions for the current user
- Get a session's full message history
- Rename or delete sessions

Writes are leader-only; followers transparently proxy via `LeaderRouter`.

## Retention and Compaction

Long-running chats are kept manageable by two mechanisms:

- **TTL sweep** — sessions older than a configured TTL are dropped.
- **Per-user session cap** — each user's oldest sessions are evicted beyond a configured count.
- **LLM-driven compaction** — `MemoryCompactor` replaces the oldest block of turns with a one-shot LLM summary once a session crosses a compaction threshold, preserving the most recent turns verbatim.
