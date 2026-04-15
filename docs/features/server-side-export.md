# Server-Side Export

Mium can render chat result tables into XLSX, PDF, or PPTX server-side using bundled Python scripts (`openpyxl`, `reportlab`, `python-pptx`). The output is envelope-encrypted before it touches durable storage and decrypted only on the way out to the client.

## Why Server-Side

The chat UI also has client-side exporters that run inside the browser via `xlsx.js`, `jsPDF`, and `pptxgenjs` — fine for small results. The server-side path is the right tool when:

- The result set is large and you don't want to push it through the browser's JS heap.
- You want a consistent rendering across browsers (Python libraries are deterministic).
- You want the encrypted-at-rest audit trail provided by the [Temp File Storage](temp-file-storage.md) layer.

## The Endpoint

```
POST /admin/api/export
{
  "format":  "xlsx" | "pdf" | "pptx",
  "title":   "Quarterly revenue",
  "sql":     "SELECT …",
  "columns": [{"name":"country","type":"VARCHAR"}, …],
  "rows":    [["KR", 12345], ["US", 67890], …]
}
```

The response is a streaming download with `Content-Type` and `Content-Disposition` set for the chosen format.

## Pipeline

1. Serialize the request to JSON.
2. Spawn `python3 <script> --output <plaintextPath>`, piping the JSON via stdin. Scripts live under `${MIUM_HOME}/python/`.
3. When the subprocess exits 0, hand the plaintext path to the active TempFileStore — it envelope-encrypts under `mium-tempfile`, persists the ciphertext to the configured backend (local FS or S3-compatible), and deletes the plaintext.
4. The endpoint streams the decrypted bytes to the HTTP response and tells the TempFileStore to delete the ciphertext after the response is written. Successful exports leave nothing behind.

## Encryption Properties

- Plaintext on disk only exists between the subprocess writing and the encrypt step that follows it (single function call, ~tens of ms).
- The ciphertext blob has no XLSX / PDF / PPTX magic — `file(1)` reports it as opaque `data`. The bytes are AES-256-GCM with a per-file DEK, the DEK wrapped under the KMS `mium-tempfile` KEK.
- Decryption happens only in-process at download time; the response body never re-lands on disk.

## Failure Modes

| Cause | Behaviour |
|---|---|
| Python missing or import error (e.g. `openpyxl` not bundled) | Endpoint returns `500` with the Python stderr trimmed; nothing is persisted. |
| Subprocess exits non-zero | Same — partial plaintext file (if any) is best-effort deleted. |
| TempFileStore PUT fails | Plaintext deleted, error returned, no orphan ciphertext. |
| Stream-decrypt fails | Ciphertext is still cleaned up in the `finally` block. |

## Bundled Python

`./package.sh` bundles `openpyxl`, `reportlab`, and `python-pptx` into `lib/python/` of the dist using `pip install --target`. The runtime needs the `python3` on `PATH` to be the same major.minor version as the one that built the bundle — wheel ABI tags must match. Operators on a host where `pip3` and `python3` resolve to different versions need to either pin both to the same Python or rebuild `lib/python/` against the runtime Python.

## Format Notes

- **XLSX** — straight `openpyxl` workbook.
- **PDF** — `reportlab`; auto-switches to landscape when columns exceed 5; row pagination handled by the script.
- **PPTX** — `python-pptx`; rows are capped (default 20) per slide with a "Showing N of M" footer to keep the deck readable.
