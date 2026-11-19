# `pskit.yml` reference

File names searched: `pskit.yml`, `pskit.yaml`, `serverless_kit.yml`, `serverless_kit.yaml` (or `-c FILE`). Unknown keys are errors.

## Top level

| Key | Notes |
|---|---|
| `service` | required; 2–32 chars, lowercase letters/digits/`-`, starts with a letter. Renaming a deployed service creates new resources |
| `provider` | required; mapping or shorthand `provider: aws` |
| `functions` | name → function (name: letter first, letters/digits/`-`/`_`, ≤ 63) |
| `workflows` | name → workflow ([workflows.md](workflows.md)) |
| `resources` | name → resource ([resources.md](resources.md)) |
| `package` | bundling |
| `hooks` | shell commands around package/deploy/remove |
| `custom` | free values for `${self:custom.x}` |
| `providers` | per-cloud overrides |
| `stages` | per-stage overrides |
| `extends` | inherit file(s) |

## provider

| Key | Default | Notes |
|---|---|---|
| `name` | – | `aws azure gcp alibaba oracle ibm digitalocean tencent huawei cloudflare onprem` |
| `region` | provider default | |
| `stage` | `dev` | `[a-z][a-z0-9-]{0,15}`, `--stage` overrides |
| `runtime` | python3.12 where supported, else newest below | `python3.9`–`python3.14`, `nodejs18/20/22/24`. Leave unset for portability |
| `memory` | 256 | 64–32768 MB |
| `timeout` | 30 | 1–3600 s, ≤ provider max |
| `architecture` | `x86_64` | or `arm64` |
| `environment` | `{}` | values stringified |
| `tags` | `{}` | + `pskit-service`, `pskit-stage`, `managed-by` |
| `iam.statements` | `[]` | AWS-style `{effect, actions, resources}`; ignored with warning where unmapped |
| `iam.role` | – | existing role / agency |
| `log_retention_days` | 14 | |
| `monitoring` | on | `{logs, metrics}` or bool |
| `options` | `{}` | provider-specific ([providers-and-troubleshooting.md](providers-and-troubleshooting.md)) |
| `state` | local `.pskit/<provider>-<stage>/terraform.tfstate` | `{backend: s3|gcs|azurerm|..., config: {...}}` |
| `terraform` | | `binary`, `required_version`, `provider_version`, `extra` (raw TF JSON deep-merged) |

## functions.<name>

```yaml
functions:
  create-order:
    handler: app.orders.create        # required
    description: Creates an order
    enabled: true                     # false = keep in file, skip deploy
    runtime: python3.12
    memory: 512
    timeout: 20
    architecture: arm64
    concurrency: 10
    environment: {TABLE: orders}      # merged over provider.environment
    tags: {team: checkout}
    iam: {statements: [{actions: ["dynamodb:PutItem"], resources: ["${tf:aws_dynamodb_table.orders.arn}"]}]}
    events: [{http: POST /orders}]    # none = invoke-only
    dlq: {queue: orders-dlq, max_retries: 3}   # queue triggers; aws, azure, gcp, onprem
    monitoring: {logs: true, metrics: false}
    package_type: image               # zip | image | auto (Azure: same for every function)
    options: {}                       # merged into the provider's function resource
```

Deployed names: `<service>-<stage>-<function>` (lowercased, sanitized, hashed if too long).

## Events (triggers)

Each item is a single-key mapping.

### http

```yaml
- http: GET /users/{id}
- http: /health                       # GET
- http:
    path: /users
    method: POST                      # GET POST PUT PATCH DELETE HEAD OPTIONS ANY
    cors: true                        # or {origins, methods, headers, expose_headers, allow_credentials, max_age}
    auth: none                        # none | iam | jwt | key
    authorizer: {issuer: ..., audience: [...]}   # jwt (AWS only)
```

| auth | Providers |
|---|---|
| none | all |
| iam | aws, gcp, huawei |
| jwt | aws only |
| key | azure (function key: `x-functions-key` header or `?code=`), huawei |

Routing is done by the pskit runtime too, so params and 405 work on gateway-less clouds. CORS preflights answered by the runtime everywhere. `allow_credentials: true` echoes the Origin — list explicit `origins`.

### schedule

```yaml
- schedule: 5 minutes
- schedule: "0 3 * * *"
- schedule:
    rate: 1 hour                      # or cron: "30 8 * * 1-5" (exactly one)
    input: {report: daily}            # -> event.data
    timezone: Europe/Berlin           # aws, gcp, alibaba(cron); azure ignores (UTC)
    enabled: true
    name: morning-report
```

### queue (aws SQS, gcp Pub/Sub, azure Storage Queue, onprem)

```yaml
- queue: jobs
- queue:
    name: jobs                        # or resource: jobs
    existing: false
    batch_size: 10                    # aws
    batch_window: 0                   # aws
    visibility_timeout: 120
    dead_letter: jobs-dlq             # aws; azure uses <queue>-poison
    max_receive: 5
    fifo: false                       # aws
```

### storage

```yaml
- storage:
    bucket: uploads                   # or resource: uploads
    existing: false
    events: [created, deleted]        # each cloud allows its own set; azure: created, updated
    prefix: incoming/
    suffix: .csv
```

Supported: aws S3, gcp GCS, azure Blob, alibaba OSS, oracle, tencent COS (`name-APPID`), huawei OBS, onprem directory/bucket.

### topic

```yaml
- topic: {name: order-events, existing: false, filter: {type: [created]}}   # filter: AWS SNS; onprem exact match
```

aws SNS, gcp Pub/Sub, azure Service Bus, alibaba MNS, huawei SMN, onprem.

### smtp (on-prem)

```yaml
- smtp: {resource: inbound-mail}
```

### Trigger matrix

| Trigger | aws | gcp | azure | alibaba | oracle | ibm | do | tencent | huawei | cloudflare | onprem |
|---|---|---|---|---|---|---|---|---|---|---|---|
| http | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓¹ | ✓ | ✓ |
| schedule | ✓ | ✓ | ✓ | ✓ | – | – | ✓ | ✓ | ✓ | ✓ | ✓ |
| queue | ✓ | ✓ | ✓ | – | – | – | – | – | – | – | ✓ |
| storage | ✓ | ✓ | ✓ | ✓ | ✓ | – | – | ✓ | ✓ | – | ✓ |
| topic | ✓ | ✓ | ✓ | ✓ | – | – | – | – | ✓ | – | ✓ |
| smtp | – | – | – | – | – | – | – | – | – | – | ✓ |

¹ needs `provider.options.apig_instance_id`.

## package

| Key | Default | Notes |
|---|---|---|
| `type` | `zip` | `zip`, `image` (AWS ECR / Azure ACR, needs Docker), `auto` |
| `image.repository` / `base_image` / `dockerfile_lines` / `keep` / `scan_on_push` | | image tuning; `dockerfile_lines` installs OS packages |
| `source` | `.` | bundled directory |
| `include` / `exclude` | `**/*` / `[]` | gitignore-style |
| `requirements` | `requirements.txt` | `py-serverless-kit` lines dropped |
| `pip_args` | `[]` | |
| `vendor` | provider default | force/skip pip install into bundle |

Always excluded: VCS dirs, `.pskit`, `.terraform`, venvs, caches, `node_modules`, `tests/**`, `dist`, `build`, `.env*`, tfstate, `pskit*.yml`.

Remote-build providers (GCP, Azure, Oracle, Cloudflare) receive `requirements.txt`; the others get manylinux wheels pip-installed into the zip for the target Python/arch.

## hooks

`before_package`, `after_package`, `before_deploy`, `after_deploy`, `before_remove`, `after_remove` → lists of shell commands, run in the project root with `PSKIT_SERVICE`, `PSKIT_STAGE`, `PSKIT_PROVIDER`, `PSKIT_WORK_DIR`. Non-zero exit aborts.

## Variables

| Syntax | Value |
|---|---|
| `${env:NAME}` / `${env:NAME, default}` | env var (error if unset without default) |
| `${opt:stage}` `${opt:provider}` `${opt:region}` | effective CLI values |
| `${self:dotted.path}` | any merged config value |
| `${file(path.json)}` | JSON/YAML parsed, other files as text |
| `${tf:<expr>}` | passed to Terraform as a live reference |

A value that is exactly one variable keeps its type. Other `${...}` (e.g. `${aws:username}`) stay literal. `${env:...}` resolves on the deploying machine and lands in Terraform state.

## Overrides and precedence

```yaml
providers:
  gcp:
    region: europe-west1              # shorthand for provider.region
    options: {project: my-proj}
    functions:
      thumbnails: {enabled: false}    # off on GCP only
      api: {events: [{http: GET /orders}]}   # lists replace
stages:
  prod:
    provider: {memory: 1024}
    providers:
      onprem: {state: {backend: null}}
extends: pskit.yml                    # in pskit.gcp.yml; or a list
```

Later wins: each file in the `extends` chain (parent first: own values, then its `providers.<active>`), then `stages.<active>`, then `stages.<active>.providers.<active>`, then CLI flags `--provider/--stage/--region`. Dicts deep-merge, lists replace.

## Escape hatches

| Need | Use |
|---|---|
| Function argument pskit doesn't model (VPC, layers, tracing) | `functions.<fn>.options` |
| Provider settings | `provider.options` |
| Extra cloud resources | `provider.terraform.extra` (raw TF JSON) |
| Reference them | `${tf:aws_dynamodb_table.orders.name}` in environment / iam |
