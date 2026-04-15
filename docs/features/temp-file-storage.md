# Temp File Storage

Mium renders some files server-side — currently the XLSX / PDF / PPTX exports produced by the bundled Python scripts (see [Server-Side Export](server-side-export.md)). The ciphertext for those files is held by a pluggable **TempFileStore** so operators can choose where the bytes physically land.

## Backends

Two backends ship today:

- **`local`** (default) — writes ciphertext to `mium.tempfile.local.dir` (defaults to `${mium.base.data.dir}/export`). Self-contained, no dependencies.
- **`s3`** — uploads ciphertext as objects to any S3-compatible endpoint (ShannonStore, MinIO, AWS S3) via AWS SDK v2. Credentials live in the ConnectionStore under a `tool=s3` connection.

Both backends store ciphertext only — the plaintext that the Python script wrote is envelope-encrypted under the `mium-tempfile` KMS key before it leaves the master, then the plaintext file is deleted.

## Configuration

```properties
mium.tempfile.backend         = local            # local | s3
mium.tempfile.local.dir       = ${mium.base.data.dir}/export
mium.tempfile.filename.prefix = mium-export
mium.tempfile.kms.key.id      = mium-tempfile

# S3-only — ignored when backend=local
mium.tempfile.s3.connection   = s3-default       # ConnectionStore id
mium.tempfile.s3.endpoint     = http://localhost:8000
mium.tempfile.s3.region       = us-east-1
mium.tempfile.s3.bucket       = mium-tempfile
mium.tempfile.s3.path.style   = true
```

S3 access keys never appear here — they are pulled from a `tool=s3` ConnectionStore entry referenced by id.

## Hot Reload

The active TempFileStore is wrapped by a process-level holder. The admin REST endpoints rebuild it when the configuration changes:

- `GET  /admin/api/settings/tempfile` — current configuration snapshot.
- `PUT  /admin/api/settings/tempfile` — update backend / endpoint / bucket / etc. Triggers a hot reload; the next export uses the new backend with no master restart.
- `POST /admin/api/settings/tempfile/test` — round-trip probe (PUT + GET + DELETE) against either the currently-active store or a candidate config in the request body. Powers the "Test connection" button in the UI.

## Admin UI

Settings → Administration → **Temp File Storage** exposes:

- Backend radio (Local disk / S3-compatible).
- Per-backend fields (local dir, or S3 endpoint / region / bucket / path-style + ConnectionStore id).
- "Test connection" button that issues the probe before saving.
- Save button that PUTs the new config and reloads.

The page redirects operators to the Connections page for entering S3 credentials — the setting page itself never accepts secrets.

## Lifecycle of a Single Export

1. Python renders the file to a local scratch dir (Python libs need a filesystem path).
2. The TempFileStore reads the plaintext, envelope-encrypts it under the `mium-tempfile` KMS key, persists the ciphertext (local file or S3 object), and deletes the plaintext.
3. The HTTP response handler streams the decrypted bytes to the client.
4. The store deletes the ciphertext immediately after the response is written — there is no caching layer on the export path; every export regenerates from scratch.

The only on-disk plaintext window is between Python writing and the encrypt step that follows it — typically tens of milliseconds.

## What Is NOT Wired Yet

The chat UI's "Excel로 만들어줘" buttons currently render the file client-side in the browser via `xlsx.js` / `jsPDF` / `pptxgenjs`, and never call `/admin/api/export`. So the TempFileStore is exercised today only when something explicitly POSTs to that endpoint. A planned change will route chat-driven exports through the server, surface a one-time signed download link, and let users click to actually retrieve the file from the configured backend — which makes the S3 backend useful for everyday chat-driven file generation.
