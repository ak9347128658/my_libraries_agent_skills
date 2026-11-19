---
name: py-serverless-kit
description: Build, test and deploy serverless functions with py-serverless-kit (pskit), the Terraform-based toolkit that runs one Python or Node.js handler on AWS Lambda, Azure Functions, Google Cloud Functions, Alibaba, Oracle, IBM, DigitalOcean, Tencent, Huawei, Cloudflare Workers or on-premises servers from one `pskit.yml`. Use when a project has `pskit.yml`, code imports `py_serverless_kit` / `pskit`, the user mentions pskit or pskit studio, wants to write handlers (HTTP, schedule, queue, storage, topic, SMTP), workflows/state machines, resources, deploy/validate/invoke/logs, or port an AWS Lambda / Step Functions app to another cloud.
---

# py-serverless-kit (pskit)

Write a handler once against a small provider-neutral API (`Event`, `Context`, `Response`). Describe functions, triggers, workflows and resources in `pskit.yml`. pskit renders Terraform for the chosen cloud, packages the code (the runtime is vendored automatically) and deploys. Changing `provider.name` is the migration.

- PyPI `py-serverless-kit` · import `py_serverless_kit` · CLI `pskit` · repo github.com/kavion-ai/py-serverless-kit · this skill covers **3.2.x**
- Needs Python ≥ 3.9; Terraform ≥ 1.5 (or OpenTofu) to deploy; Docker only for images, one-click on-prem services, Full VS Code and Hana.

## Two ways to drive a project — both write the same `pskit.yml`

| | **pskit studio** (web UI) | **CLI** |
|---|---|---|
| Start | `pskit studio --open` → open the printed `http://127.0.0.1:8770/#token=...` link | `pskit init my-api -p azure -t http` |
| Edit | Forms: Overview & provider, Functions wizard, Workflows diagram, Resources, Code, YAML | edit `pskit.yml` + handler files |
| Check | live validation, Problems panel, **Setup & credentials** (`#/setup`) | `pskit validate`, `pskit doctor -p <cloud>` |
| Run locally | **Test → Run locally** | `pskit invoke`, `pskit dev`, pytest |
| Deploy | **Deploy** page (Validate, Package, Plan, Deploy, Info, Remove) | `pskit deploy -s dev` |
| Observe | **Logs**, **Test → Send to <cloud>** | `pskit logs -f`, `pskit trigger`, `pskit info` |

### Answering "how do I … in studio?" questions

For any question about the studio UI (where a button is, how to create, deploy, test or configure something by clicking), **read [references/studio.md](references/studio.md) first**. It has:
- the sidebar and screen map
- a quick-answer table of click paths, grouped by task
- an index of every section of the 25 studio guide pages in [references/studio/](references/studio/)

Open the linked page for the full numbered steps.

Answer with:
1. The click path as **Sidebar page → section → button**, using the exact labels in bold.
2. What it writes to `pskit.yml`.
3. The equivalent CLI command.
4. A reminder to press **Save**: studio edits a draft until you save.

When you edit files directly instead, remember that studio rewrites `pskit.yml` on Save. Comments are dropped, key order changes, and the old file is backed up to `.pskit/studio/backups/`.

## Project layout

```
my-service/
├── pskit.yml               # the project (keep in git)
├── requirements.txt        # shared runtime deps — never list py-serverless-kit here
├── app/handlers.py         # handler: app.handlers.<fn>
├── functions/<name>/       # "own folder" functions (studio default for new functions)
│   ├── __init__.py
│   ├── handler.py          # handler: functions.<name>.handler.main
│   └── requirements.txt    # this function's own deps
├── tests/                  # excluded from bundles
├── .env                    # local-only variables (git-ignored, never bundled)
└── .pskit/                 # build output, terraform state, studio data (git-ignored)
```

Handlers are imported **as packages**: every folder on the handler path needs `__init__.py`, and code inside a function folder should use relative imports (`from .utils import x`) or full package paths, not bare sibling imports. Split a shared file into its own folder with `pskit function split <fn>` (studio: **Functions → Own folder**).

## Handler essentials

```python
from py_serverless_kit import Event, Context, Response, BatchResult, NotFound, BadRequest

def get_order(event: Event, ctx: Context):            # HTTP
    order = ORDERS.get(event.http.path_params["id"])
    if order is None:
        raise NotFound("no such order")               # -> 404 JSON on every cloud
    return order                                      # dict -> 200 JSON

def create_order(event: Event, ctx: Context):
    body = event.http.json()                          # bad/empty JSON -> automatic 400
    ctx.logger.info("creating", extra={"sku": body.get("sku")})
    return Response.json(save(body), status=201)

def worker(event: Event, ctx: Context):               # queue / topic / storage
    result = BatchResult()
    for record in event.records:
        try:
            process(record.json())
        except Exception:
            result.fail(record)                       # only failed messages retried (AWS, on-prem)
    return result

def nightly(event: Event, ctx: Context):              # schedule / invoke / workflow task
    return run_report(event.data)
```

- Return value → response: `dict/list/dataclass/pydantic` = JSON 200, `str` = text, `bytes` = octet-stream, `None` = 204, `(body, status[, headers])` Flask-style, or a `Response`.
- `HttpError` subclasses map to status codes (`BadRequest` 400 … `TooManyRequests` 429). Other exceptions → 500 with only `request_id` (traceback in logs; `PSKIT_DEBUG=1` shows it — never in prod). Non-HTTP handlers re-raise so the platform retries.
- `async def` handlers are awaited. One-argument `def h(event)` also works.
- Never parse native payloads (`event["Records"]`, `requestContext`) — use `event.http`, `event.records`, `event.data`. `event.raw` exists but isn't portable.
- Full API (Event, HttpRequest, Record, Context, Response, errors, `invoke()`, `resource()`, `get_secret()`, env vars, logging): [references/handler-api.md](references/handler-api.md).

## `pskit.yml` essentials

```yaml
service: orders                     # 2–32 chars, lowercase, starts with a letter
provider:
  name: azure                       # aws gcp azure alibaba oracle ibm digitalocean tencent huawei cloudflare onprem
  region: eastus
  stage: dev
  runtime: python3.12               # omit to let pskit pick the best supported version
  timeout: 30
  environment: {LOG_LEVEL: INFO, API_KEY: "${env:API_KEY}"}
  options: {subscription_id: "${env:ARM_SUBSCRIPTION_ID}"}
functions:
  get-order:
    handler: app.handlers.get_order
    events:
      - http: GET /orders/{id}
  worker:
    handler: app.handlers.worker
    events:
      - queue: {name: jobs, batch_size: 10}
  nightly:
    handler: app.handlers.nightly
    events:
      - schedule: {cron: "0 2 * * *", input: {report: daily}}
providers:                          # per-cloud overrides, merged only when that cloud is active
  aws: {region: us-east-1}
stages:                             # per-stage overrides
  prod: {provider: {memory: 1024}}
```

Unknown keys are rejected. Each `events` item is a **single-key** mapping. Lists in overrides **replace**, dicts deep-merge. Secrets go in `${env:VAR}` or a secret store, never literal YAML. Full schema, variables, precedence, triggers per provider, escape hatches: [references/pskit-yml.md](references/pskit-yml.md).

## The development loop

```bash
pskit doctor -p azure                        # tools + credentials for the target
pskit validate                               # schema + what this provider can't run
pskit invoke -f get-order --path /orders/1   # run one handler locally (no cloud)
pskit invoke -f worker -t queue -d '[{"id":1},{"id":2}]'
pskit dev                                    # local HTTP server, hot reload, :3000
python -m pytest                             # LocalRunner tests (references/testing.md)
pskit package                                # build zip + main.tf.json, show size vs limits
pskit deploy -s dev                          # terraform apply (--confirm to review plan)
pskit info -s dev                            # URLs, ids, console links
pskit trigger -f worker -e 0 -d '{"id":1}'   # real event to the deployed function
pskit logs -f --function worker
pskit remove -s dev                          # destroy (--keep-resources keeps data)
```

Always run `pskit validate` (for every target cloud: `-p aws`, `-p gcp`, …) after editing `pskit.yml`, and `pskit invoke`/pytest after editing handlers, before suggesting a deploy. Deploys, `remove`, and `trigger` touch real cloud resources — confirm with the user first. All commands: [references/cli.md](references/cli.md).

## Rules that prevent the common mistakes

1. **Don't add `py-serverless-kit` to `requirements.txt`** — it's vendored and such lines are stripped.
2. **Don't create `function_app.py`, `host.json` (Azure) or a root `main.py` (GCP)** — pskit generates them.
3. **Two functions can't share `METHOD /path`.** Path params are `{name}` (one segment) or `{name+}` (rest of path, last segment only); names are identifiers (`{user_id}`, not `{user-id}`).
4. **Trigger support differs per cloud.** Queue triggers exist only on aws/gcp/azure/onprem; `auth: jwt` is AWS-only; Azure has `auth: key` but no `iam`/`jwt`, no blob delete events, UTC-only timers, app-wide environment, 600 s max. `pskit validate -p <cloud>` lists what breaks; disable per cloud with `providers.<cloud>.functions.<fn>.enabled: false`.
5. **Write idempotent consumers** — redelivery is normal everywhere; return `BatchResult` instead of raising for per-message failures.
6. **Schedules**: portable `rate` (`5 minutes`) or 5-field cron, converted per cloud. Rates must divide 60 min / 24 h on cron-only clouds.
7. **Resource bindings, not hard-coded names**: declare under `resources:` and read with `resource("key")` (`.native_name`, `.url`, `.arn`, `.get(...)`). Put the *name* of a secret variable in `connection` (`password_env: DB_PASSWORD`), read `ctx.env[r["password_env"]]`.
8. **Embedded workflows must finish inside one invocation** (provider max timeout). Long `wait`s need a native engine (AWS Step Functions, GCP Workflows, Azure `engine: native`) or on-prem.
9. **Big bundles**: zip over the provider limit → `package.type: image` (or `auto`) for AWS ECR / Azure ACR; Azure images need a non-Consumption `plan_sku` (e.g. `EP1`). System packages via `package.image.dockerfile_lines`.
10. **State**: default Terraform state is local in `.pskit/`. Teams/CI need `provider.state` with a remote backend.
11. **Studio security**: it binds to 127.0.0.1 and the `#token=` link grants full project + cloud access — don't share it or bind `0.0.0.0` on untrusted networks.

## Reference files

| Need | File |
|---|---|
| Studio navigation hub: screen map, "how do I" click paths, index of every studio section | [references/studio.md](references/studio.md) |
| Studio page guides, step by step: start, new-project, overview, setup, functions, trigger-forms, workflows, resources, servers, onprem-services, replication, monitoring, cluster, load-balancers, network, firewall, placement, code, test, deploy, logs, yaml, accounts-and-team, hana, end-to-end (a full app, click by click) | [references/studio/](references/studio/) |
| Handler API: Event, Context, Response, errors, BatchResult, invoke, resource, secrets, logging | [references/handler-api.md](references/handler-api.md) |
| `pskit.yml` schema, triggers, variables, overrides, packaging, escape hatches | [references/pskit-yml.md](references/pskit-yml.md) |
| Every CLI command and flag | [references/cli.md](references/cli.md) |
| Workflows (state machines): states, choice rules, retry/catch, engines | [references/workflows.md](references/workflows.md) |
| Resources, on-prem services, secrets, publishing to queues/buckets | [references/resources.md](references/resources.md) |
| Local testing: LocalRunner, http_event, build_event, workflow tests, CI | [references/testing.md](references/testing.md) |
| Provider matrix, Azure/AWS/GCP/on-prem specifics, troubleshooting | [references/providers-and-troubleshooting.md](references/providers-and-troubleshooting.md) |
| Porting AWS Lambda + Step Functions to pskit | [references/migrate-from-aws-lambda.md](references/migrate-from-aws-lambda.md) |
| The 21 official examples and which pattern each shows | [references/examples.md](references/examples.md) |

## File storage inside handlers

pskit gives you portable triggers but deliberately no portable "put object" call on clouds. For provider-neutral file storage in handlers, pair it with **py-storebridge** (see that skill): add `py-storebridge[<extra>]` to the function's `requirements.txt`, create the client once at module level from env vars, and keep bucket/container names in `resources:` or `environment`.
