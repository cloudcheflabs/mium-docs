# Server-Side Export

Mium renders chat result tables into XLSX, PDF, or PPTX on **Workers** using bundled Python scripts (`openpyxl`, `reportlab`, `python-pptx`). The output is envelope-encrypted and uploaded to S3-compatible storage. The Master serves the download to the user via a link that stays valid for the retention window.

> In the chat UI, CSV and Markdown are produced **client-side** (inline preview / browser download); the binary formats (XLSX / PDF / PPTX) are the ones rendered server-side on a Worker and delivered via a `downloadUrl`.

## How It Works

1. User says "Excel 로 만들어줘" in chat.
2. The LLM responds with `exportFormat: "xlsx"` alongside the query result.
3. `ChatEndpoints` detects the export format and dispatches `EXECUTE_EXPORT` to a ready Worker via internal NIO.
4. The Worker runs `python3 export_xlsx.py --output <path>` (the request payload is delivered as JSON on stdin), envelope-encrypts the output under `mium-tempfile`, and PUTs the ciphertext to the configured S3 bucket.
5. The Worker returns the S3 handle to the Master.
6. The chat response includes a `downloadUrl`. The UI shows a **"Download file"** button (not an auto-download).
7. When the user clicks, the UI fetches the download endpoint with the JWT token. The Master reads from S3, decrypts, and streams to the browser. The object is retained (the link is reusable for the retention window) and reaped later by the TTL sweep — it is **not** deleted on download.

No Worker available → `503 Service Unavailable`. The Master never runs Python locally.

## The Endpoint

```
POST /api/export
```

Accepts `{format, title, sql, columns, rows}` and returns the file directly (streaming download). Used for direct API calls outside the chat flow. This path **does** delete the S3 object immediately after streaming. `format` accepts the aliases `excel` (→ xlsx) and `ppt` (→ pptx).

```
GET /api/export/download?handle=...&name=...&ct=...
```

Serves a file previously rendered via `EXECUTE_EXPORT` (the chat "Download file" button). The handle is an opaque S3 object key. The file is **not** deleted after download — the link is reusable and the object is removed by the retention sweep (`mium.tempfile.ttl.days`, default 10).

(Paths shown for the default empty `mium.admin.context.path`; a configured prefix like `/admin` is prepended.)

## Supported Formats

| Format | Python Library | Notes |
|---|---|---|
| XLSX | openpyxl | Workbook with optional SQL hint row |
| PDF | reportlab | Auto landscape when columns > 5, paginated |
| PPTX | python-pptx | Rows capped per slide with footer |

## Encryption

- Plaintext exists on the Worker's local disk only between the Python write and the encrypt step (~tens of ms).
- The S3 object contains AES-256-GCM ciphertext (no XLSX/PDF/PPTX magic visible).
- Decryption happens in-memory on the Master at download time; no plaintext re-lands on disk.
- For the direct `POST /api/export` path the S3 object is deleted immediately after the response completes; for the chat `GET /api/export/download` path the object is retained for re-download and removed by the TTL sweep.

## Python Bundle

`package.sh` bundles Python dependencies into `lib/python/` of the dist. The runtime `python3` must match the Python version used during packaging (wheel ABI tags must match).
