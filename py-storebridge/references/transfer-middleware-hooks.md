# Transfer, middleware, hooks, multipart, custom providers

## transfer(source, destination, options=None, **kwargs) -> TransferResult

Copies (or moves) **existing** objects between two `StorageClient`s, possibly on different providers. Both providers' extras must be installed.

| option | default | meaning |
|---|---|---|
| `source_key` | – | one object |
| `keys` | – | explicit list |
| `prefix` | – | everything under a folder-like prefix |
| *(none of the three)* | | whole bucket/container |
| `destination_key` | – | only with `source_key` |
| `destination_prefix` | – | prepended; with `prefix`, the source prefix is replaced by it |
| `delete_source` | `False` | move instead of copy |
| `overwrite` | `True` | |
| `skip_existing` | `False` | skip keys already on destination |
| `concurrency` | `8` | 1–64 worker threads |
| `preserve_metadata` / `preserve_visibility` | `True` | |
| `dry_run` | `False` | plan only; items get status `planned` |
| `continue_on_error` | `True` | `False` → raise `TransferError` on first failure, `.result` holds the partial result |
| `on_progress` | – | `Callable[[done:int, total:int, key:str], None]` |
| `on_item` | – | `Callable[[TransferItemResult], None]` |

Only one of `source_key` / `prefix` / `keys` may be set.

Key mapping examples:

| call | `images/a/b.png` lands at |
|---|---|
| `prefix="images/", destination_prefix="backup/images/"` | `backup/images/a/b.png` |
| `prefix="images/"` | `images/a/b.png` |
| whole bucket, `destination_prefix="mirror"` | `mirror/images/a/b.png` |

Recipe — safe migration with verification:

```python
import py_storebridge as sb

plan = sb.transfer(src, dst, prefix="invoices/", dry_run=True)
print(plan.planned, "objects,", plan.bytes_transferred, "bytes")

result = sb.transfer(
    src, dst, prefix="invoices/", skip_existing=True, concurrency=16,
    on_progress=lambda done, total, key: print(f"{done}/{total} {key}"),
)
if not result.ok:
    for item in result.items:
        if item.status == "failed":
            print(item.source_key, item.error)
```

Hooks `before:transfer` / `after:transfer` fire on the **source** client.

## Middleware (wraps `upload()` only)

Onion model. Signature: `def mw(ctx: MiddlewareContext, nxt: Callable[[], UploadResult]) -> UploadResult`. `nxt` takes **no arguments**; mutate `ctx.options` before calling it, inspect the result after.

`MiddlewareContext` fields: `options: UploadOptions`, `provider: str`, `started_at: datetime`, `metadata: dict` (scratch space), `result`.

```python
def tenant_prefix(ctx, nxt):
    if not ctx.options.key.startswith("tenants/"):
        ctx.options.key = f"tenants/{TENANT_ID}/{ctx.options.key}"
    return nxt()

def stamp(ctx, nxt):
    ctx.options.metadata = {**(ctx.options.metadata or {}), "uploaded_by": USER_ID}
    return nxt()

def timing(ctx, nxt):
    result = nxt()
    log.info("uploaded %s in %s", result.key, datetime.now(timezone.utc) - ctx.started_at)
    return result

storage.use(tenant_prefix).use(stamp).use(timing)   # registration order = outer → inner
```

Note: a key changed in middleware is not re-validated — keep it a valid key.

## Hooks

```python
storage.on("after:upload", lambda r: audit(r.key, r.size))
storage.on("error", lambda p: log.error("%s failed: %s", p["operation"], p["error"]))
storage.off("after:upload", handler)        # same function object
```

| event | payload |
|---|---|
| `before:upload` | `UploadOptions` |
| `after:upload` | `UploadResult` |
| `before:delete` / `after:delete` | `DeleteOptions` |
| `before:copy` / `before:move` | `{"source_key", "destination_key"}` |
| `after:copy` / `after:move` | `UploadResult` |
| `before:list` | `{"prefix"}` |
| `after:list` | `ListResult` |
| `before:getSignedUrl` | `{"key", "expires_in"}` |
| `after:getSignedUrl` | `{"key", "url"}` |
| `before:transfer` | `{"source", "destination", "count", "options"}` |
| `after:transfer` | `TransferResult` |
| `error` | `{"operation", "error"}` (upload, delete, list, getSignedUrl, copy, move) |

Handlers run synchronously; an exception in a handler propagates to the caller.

## Multipart

High level (whole payload in memory):

```python
storage.upload_multipart({"key": "big/data.parquet", "path": "data.parquet"}, chunk_size=16 * 1024 * 1024)
```

Low level (stream from disk without loading it all):

```python
session = storage.create_multipart_upload("big/video.mp4", "video/mp4")
parts = []
try:
    with open("video.mp4", "rb") as fh:
        n = 1
        while chunk := fh.read(8 * 1024 * 1024):          # S3 needs ≥ 5 MiB except the last part
            parts.append(storage.upload_part(upload_id=session.upload_id, key=session.key,
                                             part_number=n, body=chunk))
            n += 1
    storage.complete_multipart_upload(session, parts)
except Exception:
    storage.abort_multipart_upload(session)
    raise
```

On fallback providers (GCS, Supabase, Firebase, Appwrite) parts are buffered in the provider object's memory, so the whole upload must happen in one process.

## Browser direct upload (presigned PUT)

```python
url = storage.get_signed_url(f"uploads/{uuid4()}.csv", method="PUT", expires_in=300, content_type="text/csv")
# client: PUT url with header Content-Type: text/csv
```

## Custom provider

Subclass `BaseProvider`, set `provider_name`, implement the abstract `_` methods: `_upload, _delete, _exists, _get_metadata, _list, _get_url, _get_signed_url, _copy, _get_stream`. Override `_create_multipart_upload/_upload_part/_complete_multipart_upload/_abort_multipart_upload/_list_parts` only for native multipart (otherwise the in-memory fallback is used). `copy_via_download(options)` and `public_url(...)` are helpers. The base class adds retry, validation and MIME detection around your methods.

```python
from py_storebridge import BaseProvider, create_storage_from_provider

class MyProvider(BaseProvider):
    provider_name = "my-provider"
    def __init__(self, client, **kw):
        super().__init__(**kw)          # retry=, log_level=, default_visibility=
        self.client = client
    def _upload(self, options): ...
    # ... the other abstract methods

storage = create_storage_from_provider(MyProvider(client))
```

## Testing tip

Use the on-prem filesystem engine in tests — no service, no network:

```python
def test_upload(tmp_path):
    s = create_storage(provider="onprem", credentials={"root": str(tmp_path), "bucket": "test"})
    s.upload(key="a/b.txt", file=b"hi")
    assert s.get_buffer("a/b.txt") == b"hi"
```
