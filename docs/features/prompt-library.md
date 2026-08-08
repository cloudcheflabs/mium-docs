# Prompt Library

Mium provides a per-user prompt library — the **PromptStore** — so users can save, retrieve, and reuse prompts they care about. The store sits next to the MemoryStore, sharing the same pluggable-backend pattern.

## Model

A saved prompt has:

- An owner user id
- A short name and a longer prompt body
- Optional tags
- Created / updated timestamps

Per-user isolation is enforced at the store level — every read and write is owner-scoped, and `getPrompt` throws on an owner mismatch, so one user cannot see or modify another user's prompts. The `SYSTEM:READ_PROMPT` / `SYSTEM:WRITE_PROMPT` actions exist in the vocabulary, but because the PromptStore has no HTTP surface yet (below) nothing currently gates access through them; isolation comes from the owner-scoped store queries.

## Storage

PromptStore persists prompts in **NeorunBase** (`mium_prompt` table). Every Mium node reads and writes directly — no Mium-side replication needed. See [Storage Backends](storage-backends.md).

## Limits

The store enforces:

- **Per-user prompt cap** — once a user reaches `mium.prompt.user.maxPrompts` (default `0` = unlimited), further `createPrompt` calls are **rejected** (there is no eviction of older prompts).
- **Body size cap** — bodies above `mium.prompt.body.maxBytes` (default `65536` = 64 KiB) are rejected at write time.
- **TTL sweep** — when `mium.prompt.ttl.days` > 0, the leader's retention sweep deletes prompts older than the TTL (default `0` = disabled).

## Semantic Search (in code)

A `PromptSemanticIndex` already implements cosine-similarity search over a per-user `prompt:` namespace, embedding each prompt's `name + body + tags` with write-through upsert/delete. It is wired to the embedding subsystem, so semantic prompt lookup is implemented at the store/index level even though no user-facing surface exposes it yet.

## Future Surface

The PromptStore is initialized on the master but is **not** yet reachable over REST — there is no `/api/prompts` endpoint and no Admin UI page. Browsing / saving prompts ("save this prompt to my library", "load a saved prompt into chat") and a `save_prompt` agent-loop action so the LLM can persist a prompt on request are on the roadmap.
