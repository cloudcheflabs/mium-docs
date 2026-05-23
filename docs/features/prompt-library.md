# Prompt Library

Mium provides a per-user prompt library — the **PromptStore** — so users can save, retrieve, and reuse prompts they care about. The store sits next to the MemoryStore, sharing the same pluggable-backend pattern.

## Model

A saved prompt has:

- An owner user id
- A short name and a longer prompt body
- Optional tags
- Created / updated timestamps

Per-user isolation is enforced at every read and write — one user cannot see or modify another user's prompts unless they hold the appropriate `SYSTEM:READ_PROMPT` / `SYSTEM:WRITE_PROMPT` policy.

## Storage

PromptStore persists prompts in **NeorunBase** (`mium_prompt` table). Every Mium node reads and writes directly — no Mium-side replication needed. See [Storage Backends](storage-backends.md).

## Limits

The store enforces:

- **Per-user prompt cap** — oldest prompts are evicted beyond a configured count (`mium.prompt.max.per.user`).
- **Body size cap** — bodies above `mium.prompt.max.body.bytes` are rejected at write time.

## Future Surface

Today the PromptStore is reachable through the REST layer; an Admin UI page for browsing / saving prompts ("save this prompt to my library", "load a saved prompt into chat") is on the roadmap. The agent loop will eventually expose a `save_prompt` action so the LLM itself can persist a prompt the user asks it to remember.
