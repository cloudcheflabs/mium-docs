# Prompt Library

Mium provides a per-user prompt library — the **PromptStore** — so users can save, retrieve, and reuse prompts they care about. The store sits next to the MemoryStore, sharing the same pluggable-backend pattern.

## Model

A saved prompt has:

- An owner user id
- A short name and a longer prompt body
- Optional tags
- Created / updated timestamps

Per-user isolation is enforced at every read and write — one user cannot see or modify another user's prompts unless they hold the appropriate `MIUM:READ_PROMPT` / `MIUM:WRITE_PROMPT` policy.

## Backends

`PromptStore` is pluggable, chosen at startup by `mium.prompt.backend`:

- **`rocksdb`** (default) — leader-owned, envelope-encrypted snapshot persisted locally; replicated to followers via internal NIO sync.
- **`neorunbase`** — prompts live in the shared CCL-stack NeorunBase database (`mium_prompt` table). Every Mium node reads and writes live; NeorunBase handles durability.

Both backends expose the same API; switch by editing `mium.prompt.backend`. See [Storage Backends](storage-backends.md).

## Limits

The store enforces:

- **Per-user prompt cap** — oldest prompts are evicted beyond a configured count (`mium.prompt.max.per.user`).
- **Body size cap** — bodies above `mium.prompt.max.body.bytes` are rejected at write time.

## Future Surface

Today the PromptStore is reachable through the REST layer; an Admin UI page for browsing / saving prompts ("save this prompt to my library", "load a saved prompt into chat") is on the roadmap. The agent loop will eventually expose a `save_prompt` action so the LLM itself can persist a prompt the user asks it to remember.
