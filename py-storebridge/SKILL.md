---
name: py-storebridge
description: Write Python code that stores, reads, lists, signs, copies or migrates files with py_storebridge (PyPI `py-storebridge`), one storage API over AWS S3, Cloudflare R2, Google Cloud Storage, Azure Blob, DigitalOcean Spaces, Backblaze B2, MinIO, Supabase, Firebase, Appwrite and on-premises disk or S3-compatible servers. Use when code imports `py_storebridge`, calls `create_storage(...)`, when the user wants provider-neutral file storage or uploads, presigned/signed URLs, multipart uploads, or to transfer/migrate files, folders or whole buckets between storage providers.
---

# py_storebridge

A unified storage SDK: one `StorageClient` API, 11 backends, switch provider by changing the config. Pydantic v2 validates every config and call. Version covered: **1.1.0** (Python ≥ 3.9).

- Import name: `py_storebridge` · PyPI: `py-storebridge` (also installable as `py_storebridge`)
- Core depends only on `pydantic>=2.5`; provider SDKs are **extras** and imported lazily.

## Install the right extra

| provider value | extra | pulls in |
|---|---|---|
| `s3`, `r2`, `spaces`, `b2`, `minio` | same name | `boto3` |
| `gcs` | `gcs` | `google-cloud-storage` |
| `firebase` | `firebase` | `google-cloud-storage`, `google-auth` |
| `azure` | `azure` | `azure-storage-blob` |
| `supabase`, `appwrite` | same name | `httpx` |
| `onprem` (local disk) | none | – |
| `onprem` (MinIO/Ceph/Garage/SeaweedFS endpoint) | `onprem` | `boto3` |

```bash
pip install "py-storebridge[s3]"        # one provider
pip install "py-storebridge[s3,gcs]"    # several (needed for transfer between them)
pip install "py-storebridge[all]"
```

A missing extra raises `MissingDependencyError` (its message names the exact `pip install`).

## Core workflow

```python
import os
from py_storebridge import create_storage

storage = create_storage({
    "provider": "s3",
    "credentials": {
        "access_key_id": os.environ["AWS_ACCESS_KEY_ID"],
        "secret_access_key": os.environ["AWS_SECRET_ACCESS_KEY"],
        "region": "us-east-1",
        "bucket": "my-bucket",
    },
    "visibility": "private",                      # default for uploads
    "retry": {"max_retries": 3, "base_delay": 200, "max_delay": 10000},  # milliseconds
    "log_level": "warn",                          # debug | info | warn | error | silent
})
# keyword form works too: create_storage(provider="s3", credentials={...})

res = storage.upload(key="users/42/avatar.png", file=png_bytes)   # content_type auto-detected from key
res.key, res.url, res.etag, res.size, res.provider

storage.upload_file("local/report.pdf", key="reports/2026/report.pdf")
data  = storage.get_buffer("reports/2026/report.pdf")              # bytes
fh    = storage.get_stream("videos/intro.mp4")                     # BinaryIO
storage.download_file("reports/2026/report.pdf", "out/report.pdf")
url   = storage.get_signed_url("reports/2026/report.pdf", expires_in=900)
put   = storage.get_signed_url("uploads/raw.csv", method="PUT", expires_in=300, content_type="text/csv")
meta  = storage.get_metadata("users/42/avatar.png")                # FileMetadata
storage.exists("users/42/avatar.png")
storage.delete(key="users/42/avatar.png")
```

Every operation takes either a dict/model **or** keyword arguments: `storage.upload({"key": ..., "file": ...})` ≡ `storage.upload(key=..., file=...)`.

## Rules that prevent the common mistakes

1. **Keys**: `/` separators only, no leading or trailing `/`, no `\`, no empty/`.`/`..` segments, ≤ 1024 chars. `"/a.png"`, `"a\\b.png"`, `"dir/"` all raise `ValidationError`. Build keys with `"/".join(...)`, never `os.path.join`.
2. **`upload` needs `file` or `path`.** `file` accepts `bytes | str | pathlib.Path | BinaryIO`. `metadata` must be `dict[str, str]` (stringify numbers).
3. **Unknown fields are rejected** (`extra="forbid"`) in credentials and options. Both snake_case and the Node SDK's camelCase (`accessKeyId`, `expiresIn`) are accepted.
4. **`delete`, `copy`, `move` take keywords**: `delete(key=...)`, `copy(source_key=..., destination_key=...)`. `exists`, `get_metadata`, `get_url`, `get_buffer`, `get_stream` take the key positionally.
5. **Signed URL expiry**: `expires_in` 1 s – 7 days (604 800). Methods: `GET`, `PUT`, `DELETE`. Appwrite signed URLs are limited.
6. **`get_url` is the public URL** (custom_domain → cdn_url → provider URL). It does not make a private object readable; use `get_signed_url` for private objects.
7. **copy / move / rename work inside one provider.** To move data between providers use `transfer()` (see below).
8. **`upload_multipart` reads the whole payload into memory** before chunking. For truly huge files drive the low-level API (`create_multipart_upload` / `upload_part` / `complete_multipart_upload`) with your own chunk reader. GCS, Supabase, Firebase and Appwrite use an in-memory fallback (parts buffered in-process, single upload on complete), so parts must be uploaded from the same process.
9. **Exception names shadow builtins**: `from py_storebridge import FileNotFoundError, PermissionError` hides Python's. Prefer `import py_storebridge as sb` and catch `sb.FileNotFoundError`, or alias on import.
10. **Retries**: `FileNotFoundError`, `PermissionError`, `ValidationError` are never retried; other 4xx (except 408/429) aren't either. Exhausted retries raise `MaxRetriesExceededError` (`.attempts`, `.original_error`).
11. **Never hard-code credentials**; read from env or a secret manager. On-prem `signing_secret` must be ≥ 16 chars and shared by every process that verifies signed URLs.

## Listing

```python
page = storage.list(prefix="images/", limit=100, cursor=None, delimiter="/")
page.files            # list[FileListEntry(key, size, last_modified, etag)]
page.next_cursor      # pass back as cursor=; None when done
page.prefixes         # "folders" when delimiter is set

for entry in storage.iter_files(prefix="images/"):   # auto-paginates
    ...
deleted = storage.delete_prefix("tmp/")               # returns count
```

## Errors

```python
import py_storebridge as sb

try:
    storage.get_metadata("missing.png")
except sb.FileNotFoundError as e:        # 404, .key
    ...
except sb.PermissionError as e:          # 403
    ...
except sb.MaxRetriesExceededError as e:  # .attempts
    ...
except sb.StoreBridgeError as e:         # base: .code .status_code .provider .original_error
    ...
```

Full hierarchy and codes: [references/api.md](references/api.md#errors).

## Cross-provider transfer (unique feature)

```python
from py_storebridge import create_storage, transfer

src = create_storage(provider="s3", credentials={...})
dst = create_storage(provider="gcs", credentials={...})

transfer(src, dst, source_key="a/b.png", destination_key="x/b.png")        # one object
transfer(src, dst, keys=["a.png", "b.png"])                                # explicit list
r = transfer(src, dst, prefix="images/", destination_prefix="backup/images/",
             concurrency=8, skip_existing=True)                            # a folder
src.transfer_to(dst)                                                       # whole bucket
transfer(src, dst, prefix="uploads/", delete_source=True)                  # move
plan = transfer(src, dst, prefix="images/", dry_run=True)                  # plan only
r.ok, r.transferred, r.skipped, r.failed, r.bytes_transferred, r.items
```

Pick **at most one** scope: `source_key`, `keys` or `prefix` (none = whole bucket). `destination_key` only with `source_key`. Details, progress callbacks and failure handling: [references/transfer-middleware-hooks.md](references/transfer-middleware-hooks.md).

## Where to look next

| Need | File |
|---|---|
| Credentials for each of the 11 providers (fields, defaults, on-prem modes) | [references/providers.md](references/providers.md) |
| Every method signature, option model, result model, error class | [references/api.md](references/api.md) |
| `transfer()`, middleware, hooks, multipart, custom provider, recipes | [references/transfer-middleware-hooks.md](references/transfer-middleware-hooks.md) |

## Using it inside py-serverless-kit (pskit) functions

py_storebridge pairs well with pskit handlers: keep storage provider-neutral the same way pskit keeps handlers cloud-neutral. Add `py-storebridge[<extra>]` to the function's `requirements.txt`, build the client **once at module level** (reused across warm invocations), and read credentials from env vars or `get_secret()`. Choosing the provider from config means a pskit `provider.name` switch needs no storage code change:

```python
import json
import os
from py_storebridge import create_storage

STORAGE = create_storage({
    "provider": os.environ.get("STORAGE_PROVIDER", "onprem"),
    "credentials": json.loads(os.environ["STORAGE_CREDENTIALS"]),
})
```
