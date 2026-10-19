# pdf-md-conversion

REST API service for converting PDF to Markdown (text, tables, images, OCR scans).
Consumed as a remote API by `apps/api`. Default port **8379**.

The converter only processes PDFs and hands the results over via the API response:

- `markdown` contains the Markdown output.
- `images[].data_base64` contains image/scan bytes for the consumer to store.
- `/public/...` URLs in Markdown are relative URLs that the consumer is
  expected to serve, not files served by the converter.

The converter does not persist conversion artifacts to disk. The consumer
(`monorepo-docs-agent/apps/api`) stores and serves those files. The `DATA_DIR`
folder (default `./data`) stores the async job registry (`jobs.sqlite`).

> Auth: set `PDF_MD_CONVERTER_API_KEY` (Bearer) when sharing/deploying the
> service; `/v1/health` is always open. When the key is set, every other
> endpoint requires a token.

## How to run

Local:

```powershell
Copy-Item .env.example .env   # set PDF_MD_CONVERTER_API_KEY if auth is needed
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e .
.\dev.ps1   # http://127.0.0.1:8379
```

Docker:

```powershell
Copy-Item .env.example .env
docker compose up --build -d
curl.exe http://127.0.0.1:8379/v1/health
```

## API contract

Per-endpoint details (request + response examples): [`README_API_CONTRACT.md`](README_API_CONTRACT.md).

Base path `/v1`:

| Endpoint | Description |
|---|---|
| `GET /v1/health` | Health check (no auth) |
| `POST /v1/convert` | Synchronous conversion (small/medium PDFs) |
| `POST /v1/convert/jobs` | Async conversion → `202` (large PDFs) |
| `GET /v1/convert/jobs/{job_id}` | Job status + progress + result |
| `GET /v1/documents` | Legacy stub; always `{"documents": []}` |
| `GET /v1/download/{doc_id}` | Legacy stub; always `404` |
| `DELETE /v1/documents/{doc_id}` | Legacy stub; always `404` |

Quick example:

```powershell
curl.exe -X POST "http://127.0.0.1:8379/v1/convert" `
  -H "Authorization: Bearer $env:PDF_MD_CONVERTER_API_KEY" `
  -F "file=@document.pdf" -o resp.json
```

Notes:

- Rate limits: `POST /v1/convert` 10 req/min, `POST /v1/convert/jobs` 20 req/min (429 when exceeded).
- `PUBLIC_TTL_MINUTES` determines the `expires_at` handed to the consumer.
- The consumer must store `markdown` and decode `images[].data_base64`.
- `data_base64` increases the response size by roughly 33% over the original image bytes.
- PDF only. Borderless tables are often not detected. For 100+ page PDFs with `ocr_images=true`, use the async path.
