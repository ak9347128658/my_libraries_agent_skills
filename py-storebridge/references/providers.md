# Provider credentials (py_storebridge 1.1.0)

Shared top-level config for every provider:

| key | default | notes |
|---|---|---|
| `provider` | – | `s3` `r2` `gcs` `azure` `spaces` `b2` `minio` `supabase` `firebase` `appwrite` `onprem` |
| `credentials` | – | provider-specific, below (on-prem may be omitted) |
| `visibility` | `"private"` | `"public"` or `"private"`; per-upload `visibility=` overrides |
| `retry` | `{max_retries:3, base_delay:200, max_delay:10000, retryable_status_codes:[408,429,500,502,503,504]}` | delays in **ms**; `max_delay >= base_delay`; `max_retries` 0–50 |
| `log_level` | `"warn"` | `debug` `info` `warn` `error` `silent` |

Every provider also accepts optional `cdn_url` and `custom_domain` (used by `get_url`, `custom_domain` wins). camelCase aliases are accepted for all fields. Unknown fields raise `ValidationError`.

## s3 — AWS S3 (or any S3 API via `endpoint`)

| field | required | notes |
|---|---|---|
| `access_key_id`, `secret_access_key` | yes | |
| `region`, `bucket` | yes | |
| `endpoint` | no | custom S3-compatible endpoint |
| `force_path_style` | no | |
| `session_token` | no | temporary STS credentials |

## r2 — Cloudflare R2
`account_id`, `access_key_id`, `secret_access_key`, `bucket` (all required). Public URLs usually need `cdn_url` (e.g. `https://pub-xxx.r2.dev`) or a custom domain.

## gcs — Google Cloud Storage
| field | required | notes |
|---|---|---|
| `project_id`, `bucket` | yes | |
| `credentials` | one of | `{client_email, private_key, ...}` (extra keys allowed) |
| `service_account` | one of | full service-account JSON dict |
| `key_filename` | one of | path to a service-account JSON file |

If none is given the Google SDK falls back to Application Default Credentials.

## azure — Azure Blob Storage
`account_name`, `container_name` required, plus **`account_key` or `connection_string`** (validator rejects neither). Multipart uses native block blobs. ACL/visibility is limited (container-level on Azure).

## spaces — DigitalOcean Spaces
`access_key_id`, `secret_access_key`, `region` (e.g. `nyc3`), `bucket`; optional `endpoint`.

## b2 — Backblaze B2 (S3 API)
`application_key_id`, `application_key`, `bucket`; `region` default `us-west-004`; optional `endpoint`.

## minio — MinIO
| field | default |
|---|---|
| `end_point` (host only, no scheme) | required |
| `access_key`, `secret_key`, `bucket` | required |
| `port` | `9000` |
| `use_ssl` | `True` |
| `region` | `us-east-1` |

## supabase — Supabase Storage
`url` (must start with http(s)://, trailing `/` stripped), `service_role_key`, `bucket`. Multipart = in-process fallback.

## firebase — Firebase Storage
`project_id`, `bucket` (e.g. `my-project.appspot.com`), plus `service_account` (dict) or `key_filename`. Multipart = in-process fallback.

## appwrite — Appwrite Storage
`endpoint` (e.g. `https://cloud.appwrite.io/v1`), `project_id`, `api_key`, `bucket_id`. Appwrite IDs can't contain `/`, so the file ID is a stable SHA-256 of the key and the original key is kept as the file name — `users/profile.png` still works as a key. Signed URLs are limited.

## onprem — local disk, NAS, or your own S3-compatible service

An empty credentials object is valid: files go to `./.py_storebridge/` on local disk.

| field | default | meaning |
|---|---|---|
| `bucket` | `"default"` | 3–63 chars, lowercase letters/digits/`.`/`-` |
| `root` | `.py_storebridge` | disk directory (filesystem engine, MinIO data, `engine.json`) |
| `engine` | `"auto"` | `auto` · `filesystem` (never installs anything; air-gapped) · `minio` · `s3` |
| `endpoint` | – | `http(s)://host:port` of an existing MinIO/Ceph RGW/Garage/SeaweedFS |
| `end_point`, `port`, `use_ssl` | –, –, `False` | alternative host/port form |
| `access_key`, `secret_key` | – | also accepts `access_key_id` / `secret_access_key` |
| `force_path_style` | `True` | |
| `supports_acl` | `False` | |
| `create_bucket` | `True` | create the bucket if missing |
| `signing_secret` | generated | ≥ 16 chars. **Set it yourself** when several processes verify signed URLs |
| `base_url` | – | public base URL for links |
| `serve` | `False` | run an embedded HTTP file server so browsers can fetch public files and signed URLs |
| `host` / `serve_port` | `127.0.0.1` / `8787` | embedded server bind address |
| `cors_origin` | – | single CORS origin for the embedded server |
| `max_upload_bytes` | – | cap for signed PUT uploads |
| `install` | `False` | start MinIO on localhost if nothing is listening |
| `install_method` | `"auto"` | `auto` (binary on PATH, else Docker) · `docker` · `binary` · `none` |
| `allow_download` | `False` | allow fetching the official MinIO binary |
| `minio_image` | `minio/minio:latest` | Docker image |
| `minio_binary_url`, `data_dir`, `console_port`, `container_name` | – | MinIO bootstrap tuning |
| `probe_timeout` / `startup_timeout` | `1.5` / `30` s | |

Decision table:

| Situation | Set |
|---|---|
| No storage service installed | nothing (or `engine: "filesystem"`), optionally `root` |
| Browsers need HTTP links | `serve: True` (+ `signing_secret`); public objects readable, private need signed URLs |
| Existing MinIO/Ceph/Garage/SeaweedFS | `endpoint`, `access_key`, `secret_key` |
| MinIO binary/Docker present but not running | `install: True` |
| Download MinIO if absent | `install: True, allow_download: True` |
| Air-gapped | `engine: "filesystem"` |

Notes: `install` only starts a service on **localhost**; a remote endpoint that is down is an error (no silent fallback to disk). Generated MinIO credentials are written to `{root}/engine.json` (mode 0600 on Unix). Call `storage.provider.close()` to stop the embedded file server.

```python
storage = create_storage({
    "provider": "onprem",
    "credentials": {
        "bucket": "app",
        "root": "/var/lib/py_storebridge",
        "serve": True,
        "serve_port": 8787,
        "signing_secret": os.environ["STORAGE_SIGNING_SECRET"],
    },
})
```

## Feature matrix

| Feature | S3 | R2 | GCS | Azure | Spaces | B2 | MinIO | Supabase | Firebase | Appwrite | On-prem |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Signed URLs | yes | yes | yes | yes | yes | yes | yes | yes | yes | limited | yes |
| Multipart | native | native | fallback | native blocks | native | native | native | fallback | fallback | fallback | disk or native |
| ACL / visibility | yes | limited | yes | limited | yes | limited | limited | yes | yes | yes | yes |

Upload, delete, exists, metadata, list, copy/move, stream upload, CDN/custom domain and transfer work on all 11.
