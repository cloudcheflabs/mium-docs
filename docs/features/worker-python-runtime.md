# Worker Python Runtime

Mium runs **two kinds of Python on Workers**:

- **Short-lived subprocesses** for server-side **file rendering** — XLSX, PDF, PPTX exports.
- **Long-running daemon processes** for **embedding inference** — text (BAAI/bge-m3) and multimodal (CLIP).

Both paths land on Workers, never on Masters. Masters dispatch through internal NIO opcodes (`EXECUTE_EXPORT`, `EXECUTE_EMBED`) and Workers spawn the Python they need.

## Why Worker, Not Master

Putting Python on Workers — not Masters — buys three things:

1. **Predictable Master memory.** Embedding daemons hold ~5 GB of torch weights in RAM. Sharing that machine with a leader Master that owns IAM, KMS, and ConnectionStore would couple two very different RAM curves.
2. **Horizontal scale for free.** Adding Workers adds embedding throughput and export throughput. Masters scale for HA, not throughput.
3. **No double-loading on multi-Master HA.** If three Masters all loaded CLIP, you'd pay ~15 GB just for redundancy. With Workers owning Python, the daemon is loaded once per Worker — orthogonal to Master count.

If no Worker is ready, the Master returns `503 Service Unavailable` for chat exports and embedding-dependent features. The Master never falls back to running Python locally.

## Path A — Short-Lived Export Subprocess

Used by `EXECUTE_EXPORT` for chat-driven file renders.

| Format | Script | Library |
|---|---|---|
| XLSX | `export_xlsx.py` | openpyxl |
| PDF | `export_pdf.py` | reportlab |
| PPTX | `export_pptx.py` | python-pptx |

### Flow

1. Chat turn produces a result table; the LLM emits `exportFormat: "xlsx"`.
2. The Master sends `EXECUTE_EXPORT {format, title, sql, columns, rows}` to a ready Worker.
3. The Worker invokes `python3 export_<format>.py` via `PythonProcessRunner`, piping the spec on stdin.
4. The script writes the rendered file bytes to stdout.
5. The Worker envelope-encrypts the bytes under the `mium-tempfile` KMS key and PUTs the ciphertext to S3.
6. The Worker returns the S3 handle to the Master; the chat response carries a `downloadUrl`.

The plaintext file lives on the Worker's disk only between the script's write and the encrypt step (~tens of milliseconds). The Master's download endpoint streams from S3, decrypts in memory, and deletes the S3 object after the response completes.

### Why Subprocess (Not Daemon)

Export jobs are infrequent and CPU-bound on a single render — there's nothing to amortise. The startup cost of `python3` plus loading openpyxl is well under a second, far below the typical render time of an XLSX with several thousand rows. A daemon would just complicate the protocol.

### Dependency Bundle

Wheels are pre-bundled inside the distribution at `lib/python/`:

```
lib/python/
├── openpyxl/
├── reportlab/
├── pptx/
└── ...
```

`PythonProcessRunner` sets `PYTHONPATH=lib/python` on the subprocess. The runtime `python3` interpreter must match the Python ABI the wheels were built against — the distribution ships wheels for Python 3.9–3.12.

For full details of the export pipeline — including the encryption boundary and the download flow — see [Server-Side Export](server-side-export.md) and [Temp File Storage](temp-file-storage.md).

## Path B — Long-Running Embedding Daemon

Used by `EXECUTE_EMBED` for text and multimodal embedding inference. The daemon stays alive between calls so the ~5–10 s model load amortises across requests.

### Two Slots, Many Models

The Worker runs **at most two daemons concurrently**, one per slot:

| Slot | Script | Default Model | Dim | Modality |
|---|---|---|---|---|
| Text | `bge_embedding_server.py` | `BAAI/bge-m3` | 1024 | Text only |
| Multimodal | `clip_embedding_server.py` | `clip-ViT-B-32` | 512 | Text + image (shared embedding space) |

The slots exist because text-only models reject image input and CLIP's text encoder is weaker than dedicated text encoders for retrieval. Within each slot the actual model identifier is dynamic — switching `BAAI/bge-m3` for `sentence-transformers/all-MiniLM-L6-v2` is just a different `MIUM_BGE_MODEL` env var on the same daemon script.

Why two encoders rather than one:

- **Korean + English chat retrieval** lands on the text slot — `bge-m3` is SOTA for mixed-language corpora.
- **Image retrieval and cross-modal** (text query → image hit and vice versa) lands on the CLIP slot — only encoders trained jointly produce a shared embedding space.

### Lazy Start

Daemons start **on first call**, not at Worker boot. A Worker that never receives `EXECUTE_EMBED` traffic never burns the ~5 GB it would take to load both models. Operators that want eager startup can warm the slots by sending one tiny `embedText` and one `embedImage` during deployment validation.

### Hot Model Swap

Each `embedText` / `embedImage` call carries the desired model identifier:

- If the slot's daemon was started with the **same** identifier, the daemon is reused (~tens of ms per call).
- If the identifier **changed** (an admin switched models in Settings → Embedding), the running daemon is shut down and a fresh one is spawned. That's the one-time ~5–10 s model-load cost — paid lazily on the first call after the change.

The swap is internal — callers don't see a different shape, just a longer first call after a model change.

### Protocol — Line-Delimited JSON Over stdin/stdout

Java's `EmbeddingDaemon` writes one JSON request per line to the daemon's stdin and reads one JSON response per line from stdout:

```json
// request
{"id": "12", "op": "embed", "texts": ["hello world", "foo bar"]}
{"id": "13", "op": "embed_text",  "texts": ["a cat on a mat"]}
{"id": "14", "op": "embed_image", "images_b64": ["<base64 PNG/JPEG>"]}
{"id": "15", "op": "ping"}

// response
{"id": "12", "ok": true, "vectors": [[…], […]]}
{"id": "15", "ok": true, "pong": true}
{"id": "16", "ok": false, "error": "OOM during encode"}

// startup banner
{"id": "_ready", "ok": true, "model": "BAAI/bge-m3", "dim": 1024}
```

Per-request errors do **not** kill the daemon — the Java side reads the next response and surfaces the error to the caller. Fatal load errors (model download failed, OOM at startup) exit the daemon non-zero and the Java side's daemon manager restarts the process with backoff.

### What the Embeddings Are Used For

- **Chat memory recall** — semantic retrieval over the user's past chat turns (text slot).
- **Prompt-library recall** — find saved prompts by intent rather than substring (text slot).
- **Code-generation few-shot pool** — pick the most similar Ontul SDK examples for the LLM prompt (text slot).
- **Image search and cross-modal retrieval** — query images by text or vice versa (CLIP slot). Image inputs are base64-encoded PNG/JPEG bytes; the daemon's vision tower decides format from magic header.

Vectors are persisted in NeorunBase via the `mium_embedding` table. The `mium.embedding.dim` config must match the active model's dimension at write time — `WorkerEmbeddingBackend` rejects mismatched vectors at upsert.

### Liveness

Mium exposes a `ping` round-trip per daemon for health checks. The list of currently-loaded `(slot, model)` pairs is available to operators through the Topology / Embedding settings page.

### Dependencies

Embedding daemons need a heavier Python environment than the export scripts — `sentence-transformers`, `torch`, `transformers`, plus their CUDA / MPS bindings if you want GPU acceleration. These are **not** bundled in `lib/python/` because their wheel size and platform variability would balloon the distribution. Operators install them once on the Worker host:

```bash
pip3 install sentence-transformers torch transformers Pillow
```

The `mium.python.exe` system property (or `MIUM_PYTHON_EXE` env) lets you point the runner at a virtualenv or pyenv interpreter that already has these.

The `MIUM_HF_HOME` env can pin the HuggingFace cache directory — useful when model weights are pre-staged for air-gapped installs.

## Both Paths in One Place

| | Export | Embedding |
|---|---|---|
| Trigger | `EXECUTE_EXPORT` opcode | `EXECUTE_EMBED` opcode |
| Java side | `PythonProcessRunner.run(...)` | `EmbeddingDaemon.embedTexts(...)` |
| Process model | One-shot subprocess | Long-running daemon |
| Wire format | Stdin bytes / stdout bytes | Line-delimited JSON |
| Lifetime | Per request | Until model swap or Worker shutdown |
| Failure mode | Returns ciphertext to Master via S3 | Wraps and propagates as `Exception` to Master |
| Python deps | Bundled in `lib/python/` | Operator-installed (`sentence-transformers`, `torch`, …) |

Both paths run only on Workers. The Master never spawns Python — by design.
