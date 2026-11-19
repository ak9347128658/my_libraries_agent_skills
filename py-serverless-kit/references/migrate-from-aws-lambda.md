# Migrating AWS Lambda + Step Functions to pskit

Goal: move once to pskit, after which changing clouds is a `provider.name` change. Worked example: `examples/13-migrate-from-aws-lambda` (`before/` raw Lambda + SAM + ASL, `after/` pskit).

## 1. Handler code

```python
# before
def lambda_handler(event, context):
    body = json.loads(event.get("body") or "{}")
    order_id = event["pathParameters"]["id"]
    return {"statusCode": 200, "headers": {"Content-Type": "application/json"}, "body": json.dumps(result)}

# after
from py_serverless_kit import Event, Context

def get_order(event: Event, ctx: Context):
    body = event.http.json(default={})
    order_id = event.http.path_params["id"]
    return result
```

| Lambda | pskit |
|---|---|
| `event["body"]` + `json.loads` / `isBase64Encoded` | `event.http.json()`, `.body`, `.text()` |
| `event["pathParameters"]` | `event.http.path_params` |
| `event["queryStringParameters"]` | `event.http.query` / `multi_query` |
| `event["headers"]` | `event.http.headers` (case-insensitive) |
| `requestContext.http.method` | `event.http.method` |
| `{"statusCode", "headers", "body"}` | return dict/str or `Response.json(...)`; raise `NotFound`, `BadRequest` |
| `for r in event["Records"]` | `for r in event.records` → `r.body`, `r.json()`, `r.bucket`, `r.key`, `r.action` |
| `{"batchItemFailures": [...]}` | `BatchResult` |
| `context.aws_request_id` / `get_remaining_time_in_millis()` | `ctx.request_id` / `ctx.remaining_time_ms()` |
| `boto3.client("lambda").invoke(...)` | `invoke("other-fn", payload)` |
| `print` / `logging` | `ctx.logger.info(..., extra={...})` |

Other AWS SDK calls (DynamoDB, S3…) keep working but tie the function to AWS — hide them behind your own module. For S3 file access, switching to **py-storebridge** makes the storage call portable too.

### Packaging gotchas when porting

- pskit imports the handler **as a package** (`functions.<name>.handler.main`): add `__init__.py` to the function folder and every sub-folder (`utils/`, `llm/`, …) and switch flat imports (`import lambda_utils`) to relative ones (`from . import lambda_utils`).
- During a transition, a handler can accept both shapes: `event.records` for pskit queue events and `event.data`/`event.http.json()` containing the legacy `{"Records": [...]}`.
- Native libs (OpenCV, Tesseract, LibreOffice, poppler) → `package.type: image` + `package.image.dockerfile_lines`; on headless servers use `opencv-python-headless`.

## 2. SAM / Serverless definition → pskit.yml

| SAM | pskit.yml |
|---|---|
| `HttpApi`/`Api` `Path: /orders/{id}` `Method: get` | `- http: GET /orders/{id}` |
| `Schedule: rate(5 minutes)` | `- schedule: 5 minutes` |
| `Schedule: cron(0 12 * * ? *)` | `- schedule: "0 12 * * *"` |
| `SQS` `BatchSize: 10` | `- queue: {name: jobs, batch_size: 10, existing: true}` |
| `S3` `s3:ObjectCreated:*` + suffix | `- storage: {bucket: b, events: [created], suffix: .csv, existing: true}` |
| `SNS` | `- topic: {name: t, existing: true}` |
| `Policies` | `iam.statements` |
| `MemorySize`, `Timeout`, `Runtime`, `Environment` | `memory`, `timeout`, `runtime`, `environment` |
| JWT authorizer | `auth: jwt` + `authorizer: {issuer, audience}` |

Use `existing: true` while the old stack still owns the queues/buckets/topics.

## 3. Step Functions ASL → workflows

See the mapping table in [workflows.md](workflows.md). `Retry`/`Catch` error names become Python exception class names. `Parameters`, `ResultSelector`, intrinsics → move into a small task function.

```bash
pskit workflow run order-pipeline -i @events/order.json --history
pskit workflow show order-pipeline --format asl     # compare to the old ASL
```

## 4. Deploy side by side on AWS

```bash
pskit deploy --stage migration
pskit info --stage migration
```

Switch traffic (DNS / API mapping / event source), then remove the old stack.

## 5. Move to another cloud

```bash
pskit validate --provider azure      # lists what isn't portable
```

- add provider options (`azure: options.subscription_id`, `gcp: options.project`, …);
- replace `auth: jwt` with `auth: key`/`iam` or validate tokens in the handler;
- drop `existing: true` for resources the new cloud should create;
- remove AWS-only `functions.<fn>.options` / `terraform.extra`;
- put these in `providers.<cloud>:` or a `pskit.<cloud>.yml` with `extends: pskit.yml` (see `examples/08-multi-cloud`);
- `pskit deploy --provider azure`.

Step Functions become Cloud Workflows on GCP and the embedded engine elsewhere — mind the one-invocation time limit (Azure 600 s) or use `engine: native` (Logic Apps) on Azure.
