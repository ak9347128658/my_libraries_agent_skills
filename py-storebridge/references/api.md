# py_storebridge API reference (1.1.0)

## Factories

```python
create_storage(config: dict | StorageConfig | None = None, **kwargs) -> StorageClient
create_storage_from_provider(provider: BaseProvider) -> StorageClient
parse_storage_config(data) -> StorageConfig        # validate without connecting
PROVIDERS                                          # tuple of the 11 provider names
```

`create_storage` validates first, then imports only the chosen provider module.

## StorageClient methods

| Method | Returns | Notes |
|---|---|---|
| `upload(options=None, **kw)` | `UploadResult` | runs middleware, emits `before/after:upload` |
| `upload_file(path, key=None, **kw)` | `UploadResult` | key defaults to file name |
| `upload_multipart(options=None, chunk_size=5 MiB, **kw)` | `UploadResult` | reads full payload into memory; aborts session on failure |
| `delete(options=None, **kw)` | `None` | `delete(key=...)` |
| `delete_prefix(prefix, *, continue_on_error=True)` | `int` | deleted count |
| `exists(key)` | `bool` | |
| `get_metadata(key)` | `FileMetadata` | raises `FileNotFoundError` |
| `list(options=None, **kw)` | `ListResult` | one page |
| `iter_files(prefix=None, page_size=1000)` | iterator of `FileListEntry` | all pages |
| `get_url(key)` | `str` | public/CDN URL |
| `get_signed_url(key, options=None, **kw)` | `str` | `expires_in`, `method`, `content_type`, `query_params` |
| `copy(options=None, **kw)` | `UploadResult` | same provider |
| `move(options=None, **kw)` | `UploadResult` | copy + delete, same provider |
| `rename(source_key, destination_key)` | `UploadResult` | = move |
| `get_stream(key)` | `BinaryIO` | close it when done |
| `get_buffer(key)` | `bytes` | |
| `download_file(key, destination)` | `None` | creates parent dirs |
| `create_multipart_upload(key, content_type=None)` | `MultipartUploadSession` | |
| `upload_part(options=None, **kw)` | `MultipartPartResult` | `upload_id, key, part_number (≥1), body` |
| `complete_multipart_upload(session, parts)` | `UploadResult` | |
| `abort_multipart_upload(session)` | `None` | |
| `list_parts(session)` | `list[FileListEntry]` | |
| `transfer_to(destination, options=None, **kw)` | `TransferResult` | = `transfer(self, destination, ...)` |
| `use(middleware)` | client (chainable) | wraps `upload()` only |
| `on(event, handler)` / `off(event, handler)` | client (chainable) | lifecycle hooks |
| `.provider_name`, `.provider` | | the adapter (`storage.provider.close()` for on-prem server) |

## Option models

**UploadOptions**: `key` (required), `file` *or* `path` (one required), `content_type`, `visibility` (`public`/`private`), `metadata: dict[str,str]`, `chunk_size`, `multipart`, `content_disposition`, `content_encoding`, `cache_control`, `on_progress: Callable[[uploaded:int, total:int], None]`.

**DeleteOptions**: `key`.

**ListOptions**: `prefix`, `limit` (1–10 000), `cursor`, `delimiter`.

**SignedUrlOptions**: `expires_in` (default 3600, max 604 800), `method` (`GET`|`PUT`|`DELETE`), `content_type`, `query_params: dict[str,str]`.

**CopyOptions**: `source_key`, `destination_key`, `visibility`, `metadata`, `content_type`.

**UploadPartOptions**: `upload_id`, `key`, `part_number`, `body`.

**TransferOptions**: see transfer reference.

## Result models

- **UploadResult**: `key, url, provider, etag?, size?, content_type?, metadata?`
- **FileMetadata**: `key, size, content_type, last_modified, etag?, metadata?, visibility?`
- **FileListEntry**: `key, size, last_modified, etag?`
- **ListResult**: `files, next_cursor?, prefixes?`
- **MultipartUploadSession**: `upload_id, key` · **MultipartPartResult**: `part_number, etag`
- **TransferResult**: `transferred, skipped, failed, planned, bytes_transferred, items, source_provider, destination_provider, dry_run`, property `ok` (`failed == 0`)
- **TransferItemResult**: `source_key, destination_key, status (transferred|skipped|failed|planned), size, error?, url?`

All are Pydantic models: `.model_dump()` for JSON responses.

## Helpers

`detect_mime_type(key, fallback="application/octet-stream")`, `with_retry(fn, config, provider, logger=None)`, `merge_retry_config(retry)`, `HOOK_EVENTS`, `HookSystem`, `MiddlewareChain`, `MiddlewareContext`, `BaseProvider`.

## Errors

All inherit `StoreBridgeError` (alias `StorixError`) with `.message .code .status_code .provider .original_error`.

| Class | code | status | extra attrs | retried? |
|---|---|---|---|---|
| `ValidationError` | VALIDATION_ERROR | 400 | `.field` | no |
| `MissingConfigError` (⊂ ValidationError) | VALIDATION_ERROR | 400 | `.field .provider` | no |
| `FileNotFoundError` | FILE_NOT_FOUND | 404 | `.key` | no |
| `BucketNotFoundError` | BUCKET_NOT_FOUND | 404 | `.bucket` | no (4xx) |
| `PermissionError` | PERMISSION_DENIED | 403 | | no |
| `ProviderError` | PROVIDER_ERROR | varies | | if 5xx/408/429 |
| `NetworkError` | NETWORK_ERROR | 503 | | yes |
| `MultipartUploadError` | MULTIPART_UPLOAD_ERROR | 500 | | yes |
| `MaxRetriesExceededError` | MAX_RETRIES_EXCEEDED | 503 | `.attempts` | – |
| `MissingDependencyError` | MISSING_DEPENDENCY | 500 | `.package .extra` | – |
| `UnknownMimeTypeError` | UNKNOWN_MIME_TYPE | 400 | `.key` | – |
| `TransferError` | TRANSFER_ERROR | 500 | `.result` (partial `TransferResult`) | – |

`FileNotFoundError` and `PermissionError` share names with Python builtins — import the module (`import py_storebridge as sb`) to avoid shadowing.

Mapping to HTTP in a web handler:

```python
except sb.StoreBridgeError as e:
    return {"error": e.message, "code": e.code}, e.status_code
```
