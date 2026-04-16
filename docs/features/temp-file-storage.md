# Temp File Storage

Mium renders server-side export files (XLSX / PDF / PPTX) on **Workers** and stores the encrypted ciphertext in an **S3-compatible** object store (ShannonStore, MinIO, AWS S3, etc.). The Master dispatches the render to a Worker, then serves the download to the user from S3.

## Flow

1. User asks for a file export in chat (e.g. "Excel 로 만들어줘").
2. The Master dispatches `EXECUTE_EXPORT` to a ready Worker via internal NIO.
3. The Worker runs the Python render script, envelope-encrypts the output, and PUTs it to S3.
4. The Worker returns the S3 handle to the Master.
5. The Master includes a `downloadUrl` in the chat response.
6. The user clicks the download link → `GET /admin/api/export/download?handle=...` → Master fetches from S3, decrypts, streams to the user, and deletes the S3 object.

If no Worker is available, the Master returns `503 Service Unavailable` — it does not render locally.

## S3 Configuration

```properties
mium.tempfile.filename.prefix = mium-export
mium.tempfile.kms.key.id      = mium-tempfile
mium.tempfile.s3.connection   = s3-default       # ConnectionStore id (tool=s3)
mium.tempfile.s3.endpoint     = http://localhost:8000
mium.tempfile.s3.region       = us-east-1
mium.tempfile.s3.bucket       = mium-tempfile
mium.tempfile.s3.path.style   = true
```

S3 credentials live in the ConnectionStore (`tool=s3`, `authType=access_key`), referenced by id. The bucket is auto-created on first use.

## Admin UI

Settings → Administration → **Temp File Storage** lets operators configure S3 endpoint, region, bucket, path-style, and the ConnectionStore id. A "Test connection" button performs a round-trip probe (PUT + GET + DELETE) before saving.

## Encryption

Files are envelope-encrypted under the `mium-tempfile` KMS key before upload. The S3 bucket contains only opaque ciphertext — operators with bucket access but no KMS keys cannot read the exports. Decryption happens in-memory on the Master at download time; the decrypted bytes are streamed directly to the HTTP response and never land on disk.
