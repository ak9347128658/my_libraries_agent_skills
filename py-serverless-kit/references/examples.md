# Official examples

All live in `examples/` of github.com/kavion-ai/py-serverless-kit, are tested in CI, and open in studio with `pskit studio` inside the folder. Point the user at the closest one when they start something new.

| Example | Shows | Reach for it when |
|---|---|---|
| 01-hello-http | minimal HTTP function | first project |
| 02-rest-api | CRUD, path params, `BadRequest` with `details`, `Conflict`, 201 + location, binary (`Response.binary`), redirect, cookies, CORS via `${self:custom.cors}`, AWS JWT + per-cloud override that drops it | building an API |
| 03-scheduled-jobs | rate/cron, `input`, timezones | cron jobs |
| 04-queue-worker | `BatchResult`, permanent vs transient errors, DLQ consumer, HTTP producer using `submit_queue` on-prem | queue consumers |
| 05-storage-events | created/deleted with prefix/suffix filters | file-upload pipelines |
| 06-pubsub-topic | fan-out, filters | events / notifications |
| 07-workflow-orders | every state type, retry/catch | orchestration |
| 08-multi-cloud | one `pskit.yml` with `providers:` blocks for all 11 targets, plus `pskit.<cloud>.yml` with `extends` | multi-cloud |
| 09-function-invoke | `invoke()` between functions | function-to-function |
| 10-stages-and-config | `custom`, `${file(...)}`, `${opt:stage}`, `${env:X, default}`, `stages:` overrides, remote state for prod, `stages.prod.providers.onprem` | stage config |
| 11-custom-terraform | `terraform.extra`, `${tf:...}`, remote state, hooks | extra cloud resources |
| 12-async-and-context | async handlers, logging, `remaining_time_ms()` | context usage |
| 13-migrate-from-aws-lambda | before/after Lambda + SAM + ASL | porting from AWS |
| 14-testing | pytest `LocalRunner` fixtures, workflow tests with stubs, CI validating several providers | tests / CI |
| 15-ci-cd-deploy | GitHub Actions deploy per stage | CD |
| 16-on-premises | HTTP, schedule, queue, storage, topic, workflow on one machine | on-prem |
| 17-on-premises-remote | SSH/SCP or Docker deploy to a server | remote host |
| 18-resources | managed + external resources, bindings, Redis/MinIO on-prem | resources |
| 19-custom-drivers | custom on-prem queue/storage driver; RabbitMQ/Kafka templates | custom brokers |
| 20-onprem-external-clouds | on-prem host consuming AWS/Azure/GCP/Alibaba/Cloudflare/MinIO queues & buckets | hybrid |
| 21-nodejs-typescript | Node.js/TS HTTP routes, queue worker, schedule, workflow | Node.js |

## Patterns worth copying

Reusable CORS block (02-rest-api):

```yaml
custom:
  cors: {origins: ["https://books.example.com", "http://localhost:5173"], allow_credentials: true}
functions:
  list-books:
    handler: app.handlers.list_books
    events:
      - http: {method: GET, path: /books, cors: "${self:custom.cors}"}
```

Feature that only one cloud has, with per-cloud fallback (02-rest-api):

```yaml
functions:
  admin-stats:
    handler: app.handlers.admin_stats
    events:
      - http: {method: GET, path: /admin/stats, auth: jwt,
               authorizer: {issuer: "${env:JWT_ISSUER}", audience: ["${env:JWT_AUDIENCE}"]}}
providers:
  azure:
    functions:
      admin-stats: {events: [{http: GET /admin/stats}]}   # handler checks the header itself
```

Validation error with details (02-rest-api):

```python
if errors:
    raise BadRequest("validation failed", details=errors)    # -> 400 {"error", "status", "details"}
```

Queue worker that drops poison messages but retries transient ones (04-queue-worker):

```python
for record in event.records:
    try:
        handle_job(record.json())
    except PermanentError as exc:
        ctx.logger.error("dropping message", extra={"message_id": record.id, "reason": str(exc)})
    except Exception:
        ctx.logger.exception("job failed, will retry", extra={"message_id": record.id})
        result.fail(record)
```

On-prem vs cloud producer branch (04-queue-worker):

```python
if ctx.env.get("PSKIT_ONPREM_ROOT"):
    from py_serverless_kit.runtime.onprem_host import submit_queue
    message_id = submit_queue(queue, job)
else:
    ...  # cloud SDK call behind your own function
```

Stage-specific config (10-stages-and-config):

```yaml
custom:
  table: "${self:service}-${opt:stage}-orders"
  limits: "${file(config/limits.json)}"
provider:
  memory: "${self:custom.limits.dev.memory}"
stages:
  prod:
    provider:
      memory: "${self:custom.limits.prod.memory}"
      state: {backend: s3, config: {bucket: acme-terraform-state, key: shop/prod.tfstate, region: eu-central-1}}
```
