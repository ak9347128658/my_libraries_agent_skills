# Providers and troubleshooting

## Matrix

| Provider | Tier | HTTP | Schedule | Queue | Storage | Topic | Workflows | Python | Node.js | Images |
|---|---|:-:|:-:|:-:|:-:|:-:|---|---|---|---|
| aws (Lambda) | stable | ✓ | ✓ | SQS | S3 | SNS | Step Functions | 3.9–3.13 | 18, 20, 22 | ECR |
| gcp (Cloud Functions gen2) | stable | ✓ | ✓ | Pub/Sub | GCS | Pub/Sub | Cloud Workflows | 3.9–3.13 | 18, 20, 22 | soon |
| azure (Functions) | stable | ✓ | ✓ | Storage Queue | Blob | Service Bus | embedded / Logic Apps | 3.9–3.12 | 18, 20, 22 | ACR |
| alibaba | beta | ✓ | ✓ | – | OSS | MNS | embedded | 3.9, 3.10, 3.12 | 18, 20 | soon |
| huawei | beta | ✓ | ✓ | – | OBS | SMN | embedded | 3.9, 3.10, 3.12 | 18 | soon |
| tencent | beta | ✓ | ✓ | – | COS | – | embedded | 3.9, 3.10 | 18 | soon |
| digitalocean | beta | ✓ | ✓ | – | – | – | embedded | 3.9, 3.11 | 18 | – |
| ibm (Code Engine) | beta | ✓ | – | – | – | – | embedded | 3.11 | 18, 20 | soon |
| oracle | experimental | ✓ | – | – | ✓ | – | embedded | 3.9, 3.11 | 18, 20 | always |
| cloudflare (Workers) | experimental | ✓ | ✓ | – | – | – | embedded | 3.11–3.13 | 18, 20, 22 | soon |
| onprem | experimental | ✓ | ✓ | files/drivers | dir/drivers | files | embedded | 3.9–3.14 | 18–24 | – |

`pskit providers --json` gives regions, runtimes, max timeouts, notes.

## Required options and credentials

| Provider | Must set |
|---|---|
| aws | AWS credentials (`aws configure` / SSO / env) |
| gcp | `options.project`; `gcloud auth application-default login` |
| azure | `options.subscription_id` or `ARM_SUBSCRIPTION_ID`; `az login` or `ARM_CLIENT_ID/TENANT_ID/CLIENT_SECRET` |
| alibaba | `ALICLOUD_ACCESS_KEY` / `ALICLOUD_SECRET_KEY` (or profile) |
| tencent | `TENCENTCLOUD_SECRET_ID` / `_KEY` |
| huawei | `HW_ACCESS_KEY` / `HW_SECRET_KEY`; `options.apig_instance_id` for HTTP |
| oracle | `compartment_id`, `subnet_ids`, `image` or `image_repository` (+ docker login OCIR) |
| ibm | `IC_API_KEY`, `options.resource_group` |
| digitalocean | `doctl auth init`; `options.namespace` |
| cloudflare | `CLOUDFLARE_API_TOKEN`; `options.account_id` (+ `workers_subdomain`) |
| onprem | `options.host`, `options.port`, optional `deploy: {method: local|ssh|docker, ...}`, `admin_token` |

`pskit doctor -p <provider>` checks all of it.

## Azure (one Linux Function App per service)

```yaml
provider:
  name: azure
  region: eastus
  options:
    subscription_id: ${env:ARM_SUBSCRIPTION_ID}
    resource_group: existing-rg          # optional
    plan_sku: Y1                         # Y1 Consumption (default), EP1-EP3, B1-B3, S1-S2, P1v3, P2v3
    app_insights: true                   # Log Analytics + App Insights (needed for pskit logs)
    storage_connection: OTHER_STORAGE    # app setting name for existing queues/containers elsewhere
    servicebus_namespace_id: /subscriptions/.../namespaces/my-sb
    servicebus_connection: SB_CONNECTION
```

- All functions share one app → **environment is app-wide** (conflicting keys warn, last wins); settings added in the portal are wiped on deploy — put them in `pskit.yml`.
- No per-function memory/architecture/concurrency/options; `iam.statements` ignored.
- Max timeout 600 s, one `functionTimeout` for the app (largest function timeout).
- `auth: key` → `x-functions-key` header or `?code=`; `iam`/`jwt` are errors.
- URLs: `https://<app>.azurewebsites.net/<path>` (no `/api` prefix). Invoke-only: `POST /_pskit/invoke/<fn>`. Embedded workflow: `POST /workflows/<wf>`.
- Timers are UTC. Blob triggers: created/updated only. Queue/container names 3–63 lowercase. Storage queues not FIFO; poison → `<queue>-poison`. Topic filters not applied.
- Dependencies built remotely by Oryx (first deploy takes minutes). Container images (`package.type: image`) go to ACR and need a non-Consumption plan (`plan_sku: EP1`). `package_type` must be the same for all functions.
- Don't add your own `function_app.py` or `host.json`.
- Logs: `pip install azure-monitor-query azure-identity`, then `pskit logs`. Trigger tests: `azure-storage-queue azure-storage-blob azure-servicebus` + `AZURE_STORAGE_CONNECTION_STRING`.
- Remove keeps data with `--keep-resources` (keeps storage account, Service Bus, RG).

## AWS

`iam.statements` per function (dedicated role); `auth: jwt` with `authorizer: {issuer, audience}`; API Gateway 30 s integration cap; `auth: key` unsupported on HTTP APIs; images via ECR (`package.type: image|auto`, Lambda 250 MB unzipped limit); created buckets block public access; `invoke()` needs boto3 + `lambda:InvokeFunction`.

## GCP

`options.project` required; a function with several triggers becomes several Cloud Functions (`<name>`, `<name>-queue0`, …); event functions max 540 s; a "queue" is a Pub/Sub topic with push; storage filters applied by the runtime; topic filters not applied; don't add a root `main.py`.

## On premises

`pskit deploy` starts a long-lived host (schedules, file queues, directory storage, topics, workflows survive the command). `options.deploy.method`: `local`, `ssh` (copies archive, waits for `/__pskit/health`), `docker` (`deploy.host`, `deploy.pem`, installs Docker if missing). `python: bundle` ships CPython. One process per service → distinct `options.port`. `auth` must be `none`. Set `admin_token` when binding beyond loopback; bind `host: 0.0.0.0` for remote callers. Scaling clusters (Swarm/k3s/Nomad), load balancers, placement of function groups on servers, 44 one-click services.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Neither terraform nor tofu was found on PATH` | install Terraform ≥ 1.5 / OpenTofu or set `PSKIT_TERRAFORM_BIN` |
| `pip install failed while vendoring dependencies` | vendoring providers need manylinux wheels for the target Python; pin a version with wheels, use `package.pip_args`, test with `--skip-deps` |
| Windows `[WinError 206]` / path too long | enable long paths (`LongPathsEnabled=1`) or use a short project path like `C:\src\app` |
| `unresolved variables: env:FOO` | set it or `${env:FOO, default}` |
| `each event must be a single-key mapping` | split into separate `- http: ...` / `- queue: ...` items |
| `route GET /x is used by both 'a' and 'b'` | one function per METHOD+path (`/x` = `/x/`) |
| `<provider> does not support python3.X` | remove pinned `runtime` or pick a supported one |
| child file values lose to parent | check precedence; lists replace |
| "resource already exists" after deploying from another machine | use a remote `provider.state` backend; `pskit tf -- init -migrate-state` |
| Terraform wants to update with no change | something in `.pskit/<p>-<s>/build/` changed (source, deps, manifest) |
| CORS errors | `cors: true` or mapping; explicit origins with credentials |
| 405 | method mismatch; `allow` header lists valid ones; `method: ANY` |
| 404 locally only | `pskit dev` matches exactly; check printed routes |
| 500 with only request id | find `request_id` in provider logs; `PSKIT_DEBUG=1` temporarily |
| `ModuleNotFoundError` / `attempted relative import` in a function folder | add `__init__.py` to every package on the handler path; use relative imports inside `functions/<name>/` |
| Cold starts | smaller bundle, lazy heavy imports, more memory, provisioned concurrency via `functions.<fn>.options` |
| Deploy stops: zip over limit | `--package-type image` / `package.type: auto` |

Add `-v` to any command for the full traceback.
