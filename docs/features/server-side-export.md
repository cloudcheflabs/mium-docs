# Server-Side Export

Mium renders chat result tables into XLSX, PDF, or PPTX on **Workers** using bundled Python scripts (`openpyxl`, `reportlab`, `python-pptx`). The output is envelope-encrypted and uploaded to S3-compatible storage. The Master serves the download to the user via a one-time link.

## How It Works

1. User says "Excel 로 만들어줘" in chat.
2. The LLM responds with `exportFormat: "xlsx"` alongside the query result.
3. `ChatEndpoints` detects the export format and dispatches `EXECUTE_EXPORT` to a ready Worker via internal NIO.
4. The Worker runs `python3 export_xlsx.py`, envelope-encrypts the output under `mium-tempfile`, and PUTs the ciphertext to the configured S3 bucket.
5. The Worker returns the S3 handle to the Master.
6. The chat response includes a `downloadUrl`. The UI shows a **"Download file"** button (not an auto-download).
7. When the user clicks, the UI fetches the download endpoint with the JWT token. The Master reads from S3, decrypts, streams to the browser, and deletes the S3 object.

No Worker available → `503 Service Unavailable`. The Master never runs Python locally.

## The Endpoint

```
POST /admin/api/export
```

Accepts `{format, title, sql, columns, rows}` and returns the file directly (streaming download). Used for direct API calls outside the chat flow.

```
GET /admin/api/export/download?handle=...&name=...&ct=...
```

Serves a file previously rendered via `EXECUTE_EXPORT`. The handle is an opaque S3 object key. The file is deleted after download.

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
- The S3 object is deleted immediately after the download response completes.

## Python Bundle

`package.sh` bundles Python dependencies into `lib/python/` of the dist. The runtime `python3` must match the Python version used during packaging (wheel ABI tags must match).
