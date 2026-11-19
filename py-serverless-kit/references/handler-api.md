# Handler API

Everything is importable from `py_serverless_kit` and behaves the same on every cloud. The runtime is standard-library only, Python 3.9+, and copied into every bundle.

```python
from py_serverless_kit import (
    Event, EventType, HttpRequest, Record, Headers, Context, Response,
    HttpError, BadRequest, Unauthorized, Forbidden, NotFound, MethodNotAllowed,
    Conflict, UnprocessableEntity, TooManyRequests, BatchResult, invoke,
    resource, resources, Resource, get_secret, get_secrets, put_secret, SecretNotFound, SecretError,
)
```

## Signatures

```python
def handler(event: Event, ctx: Context): ...
def handler(event: Event): ...
async def handler(event: Event, ctx: Context): ...
```

`handler:` in `pskit.yml` is `package.module.function` or `package.module:function`, relative to `package.source` (default the project root). Node.js: `src.handlers.getOrder` = export `getOrder` of `src/handlers.ts`.

## Event

| Attribute | Meaning |
|---|---|
| `type` | `"http"`, `"schedule"`, `"queue"`, `"storage"`, `"topic"`, `"invoke"`, `"workflow"` (`EventType.*`); on-prem also `smtp` |
| `provider` | `aws`, `azure`, …, or `local` under the CLI |
| `function` | logical function name |
| `http` | `HttpRequest` or `None` |
| `records` | list of `Record` (queue/topic/storage batches) |
| `data` | payload of schedule (`input`), invoke and workflow events |
| `time` | UTC datetime |
| `source` | queue/topic identifier when known |
| `raw` | native payload — not portable, avoid |

Helpers: `is_http`, `is_batch`, `json()` (HTTP body, first record body, or `data`), `for record in event`, `to_dict()` (safe to log).

## HttpRequest

| Member | Notes |
|---|---|
| `method` | upper case |
| `path`, `route` | actual path, matched template (`/users/{id}`) |
| `headers` | case-insensitive read-only; repeated headers joined with `", "` |
| `query` / `multi_query` | `dict[str,str]` (last wins) / `dict[str,list[str]]` |
| `path_params` | from `{name}` / `{name+}` |
| `body` | `bytes`, base64 already decoded |
| `source_ip`, `content_type`, `cookies` | |
| `text(encoding="utf-8")` | |
| `json(default=...)` | invalid/empty JSON raises `BadRequest` unless `default` is given |
| `form()` | `application/x-www-form-urlencoded` |

## Record

`id` (used by BatchResult), `body` (str/bytes/dict), `attributes`, `source`, storage fields `bucket`, `key`, `size`, `action` (`created`/`updated`/`deleted`), `raw`, `text()`, `json(default=...)`.

## Context

`request_id`, `function`, `provider`, `service`, `stage`, `region`, `memory_mb`, `remaining_time_ms()` (or `None`), `logger` (structured, bound to function/request_id/provider), `env` (= `os.environ`), `raw`.

## Responses

| Return | Result |
|---|---|
| `Response(...)` | as is |
| `None` | 204 |
| dict, list, numbers, dataclass, pydantic model | JSON 200 |
| `str` | text/plain 200 |
| `bytes` | application/octet-stream 200 |
| `(body, status)` / `(body, status, headers)` | Flask-style |

JSON handles datetime/date (ISO 8601), Decimal, UUID, set, and objects with `model_dump()` / `to_dict()`.

```python
Response.json({"id": 1}, status=201, headers={"x-request": "abc"})
Response.text("ok")
Response.html("<h1>Hi</h1>")
Response.binary(png_bytes, "image/png")        # base64 for gateways automatically
Response.redirect("/login", status=302)
Response.no_content()
Response.json(user).set_cookie("sid", token, max_age=3600, path="/", http_only=True, secure=True, same_site="Lax")
```

For non-HTTP events the return value goes back to the caller unchanged (next workflow state, `invoke()` result).

## Errors

| Exception | Status |
|---|---|
| `BadRequest` | 400 |
| `Unauthorized` | 401 |
| `Forbidden` | 403 |
| `NotFound` | 404 |
| `MethodNotAllowed` | 405 |
| `Conflict` | 409 |
| `UnprocessableEntity` | 422 |
| `TooManyRequests` | 429 |
| `HttpError(message, status=..., details=..., headers=...)` | any |

```python
raise UnprocessableEntity("invalid order", details={"field": "amount"})
# -> 422 {"error": "invalid order", "status": 422, "details": {"field": "amount"}}
```

Any other exception in an HTTP handler → `500 {"error": "Internal Server Error", "status": 500, "request_id": ...}`. Non-HTTP handlers re-raise so platform retry / dead-letter applies. On HTTP-only clouds (GCP, IBM, DigitalOcean, Cloudflare, Oracle) failed workflow/schedule/invoke calls return `500 {"errorType": "<ExceptionClass>", "cause": ...}`; workflow `retry`/`catch` match that class name.

## BatchResult

```python
result = BatchResult()
for record in event.records:
    try:
        process(record.json())
    except PermanentError:
        ctx.logger.error("dropping", extra={"id": record.id})   # acknowledge poison messages
    except Exception:
        result.fail(record)                                     # or result.fail(record.id)
return result                                                   # BatchResult.of(failed) also works
```

| Provider | Behaviour |
|---|---|
| aws (SQS) | `batchItemFailures` — only failed messages retried |
| onprem | host retries only failed ids, dead-letters after `max_receive` |
| local | printed as `batchItemFailures` |
| others | any failure fails the invocation → whole batch redelivered |

## invoke(): call another function

```python
result = invoke("billing", {"order_id": 42})               # waits
invoke("audit", {"order_id": 42}, asynchronous=True)       # fire and forget (AWS)
```

Resolution order: `PSKIT_ENDPOINT_<NAME>` URL (any provider) → in-process on `local`/`onprem` → AWS `lambda:InvokeFunction` (needs boto3 + IAM) → GCP authenticated HTTPS → otherwise `InvokeError` unless the endpoint variable is set. Azure invoke-only functions expose `POST /_pskit/invoke/<name>` (function key).

## resource(): bound services

```python
jobs = resource("jobs")          # KeyError listing bound names if not bound
jobs.name, jobs.type, jobs.native_name, jobs.external
jobs.connection                  # lower-cased keys
jobs.get("region", "us-east-1")  # case-insensitive with default
jobs["host"]
jobs.arn, jobs.url, jobs.id
resources()                      # all bindings of this function
```

Backed by `PSKIT_RESOURCE_<KEY>_*` env vars (`orders-db` → `ORDERS_DB`).

## Secrets

```python
password = get_secret("db-password")
webhook = get_secret("stripe", "webhook")              # field of a JSON secret
token = get_secret("optional-token", default="")       # no SecretNotFound
all_fields = get_secrets("stripe")
put_secret("last-run", "2026-10-22")                   # Vault/OpenBao and Secrets Manager
```

Store chosen by env: `PSKIT_SECRETS_DRIVER` = `env` (default: `PSKIT_SECRET_<NAME>`, then `<NAME>`), `vault` (`PSKIT_SECRETS_URL`, `PSKIT_SECRETS_TOKEN`), `aws` (Secrets Manager). `store="other"` reads `PSKIT_OTHER_*`. Cached 60 s.

## Environment variables

Injected: `PSKIT_PROVIDER`, `PSKIT_SERVICE`, `PSKIT_STAGE`, `PSKIT_REGION`, `PSKIT_FUNCTION`, `PSKIT_HANDLER` (not on Azure), `PSKIT_ONPREM_ROOT` (on-prem), `PSKIT_GCP_PROJECT` (GCP), plus `provider.environment` and `functions.<fn>.environment`.

Switches: `PSKIT_LOG_FORMAT` (`json` cloud default / `text` CLI default), `PSKIT_LOG_LEVEL` / `LOG_LEVEL` (default INFO), `PSKIT_DEBUG=1`.

## Logging

```python
ctx.logger.info("order created", extra={"order_id": 42, "amount": 9.5})
ctx.logger.exception("failed", extra={"message_id": record.id})
```

One JSON object per line with `ts, level, logger, message, function, request_id, provider` + extras. Extras colliding with `LogRecord` attributes (`name`, `message`, `args`…) get a trailing `_`. HTTP requests are auto-logged with method, path, status, duration.

## Node.js equivalent

```ts
import { NotFound, Response, BatchResult, invoke, resource } from "pskit";
import type { Context, Event } from "pskit";
export async function getOrder(event: Event, ctx: Context) {
  const order = ORDERS.get(event.http!.pathParams.id);
  if (!order) throw new NotFound("no such order");
  return order;
}
```

camelCase members (`pathParams`, `remainingTimeMs()`, `nativeName`). A service is either Python or Node.js; to mix, split into two services.
