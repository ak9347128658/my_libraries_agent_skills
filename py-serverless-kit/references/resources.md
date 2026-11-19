# Resources, services and secrets

`resources:` names the services functions use. pskit **creates** them (managed) or **attaches** existing ones (`external: true`) and hands each bound function its connection details as env vars. Handlers read them with `resource("key")`.

```yaml
resources:
  uploads: {type: storage, name: acme-uploads-1234}       # managed bucket
  jobs: {type: queue}                                     # managed queue
  billing-events:
    type: topic
    external: true
    connection: {arn: "arn:aws:sns:us-east-1:123456789012:billing-events"}
  orders-db:
    type: database                                        # database/cache/custom: always external
    external: true
    connection:
      host: ${env:ORDERS_DB_HOST, localhost}
      port: 5432
      database: orders
      password_env: ORDERS_DB_PASSWORD                    # NAME of the secret variable
    functions: [api, worker]                              # default: every user function

functions:
  worker:
    handler: app.handlers.work
    events:
      - queue: {resource: jobs, batch_size: 5}
      - storage: {resource: uploads, prefix: in/}
```

```python
from py_serverless_kit import resource

def api(event, ctx):
    uploads = resource("uploads")
    db = resource("orders-db")
    password = ctx.env[db["password_env"]]
    return {"bucket": uploads.native_name, "db": f'{db["host"]}:{db["port"]}'}
```

## Fields

| Key | Default | Notes |
|---|---|---|
| `type` | – | `storage`, `queue`, `topic`, `database`, `cache`, `custom` |
| `external` | false | pskit never creates/changes/destroys it |
| `name` | key (or tail of `connection.arn`) | native name |
| `connection` | `{}` | scalar values; keys letters/digits/`_`; `type`/`name`/`external` reserved |
| `functions` | all user functions | orchestrators excluded unless listed |
| `options` | `{}` | provider-specific (on-prem `driver`, AWS queue settings) |

Env vars per binding: `PSKIT_RESOURCES=a,b`, `PSKIT_RESOURCE_<KEY>_TYPE|NAME|EXTERNAL|<CONN_KEY>`, plus `_ARN`/`_URL` (AWS) or `_ID` (GCP). On Azure every function sees every binding (app-wide settings).

## Provider support for managed resources

| | event-backed | standalone managed | external |
|---|---|---|---|
| aws | yes | S3, SQS, SNS | yes |
| gcp | yes | bucket, Pub/Sub | yes |
| azure | yes | container, storage queue, Service Bus topic | yes |
| onprem | yes | built-in directories | yes, with drivers |
| others | where the trigger exists | no → create outside, `external: true` | yes |

AWS roles get a `pskit-resources` policy for bound storage/queue/topic resources automatically (unless you set `iam.role`).

## Publishing from handlers

pskit has **no portable "send to queue / put object" for clouds** — use the cloud SDK with the binding values (and add it to `requirements.txt`):

| Provider | queue | storage | topic |
|---|---|---|---|
| aws | `boto3` SQS (`resource(k).url`) | `boto3` S3 | `boto3` SNS (`.arn`) |
| gcp | `google-cloud-pubsub` | `google-cloud-storage` | `google-cloud-pubsub` |
| azure | `azure-storage-queue` | `azure-storage-blob` | `azure-servicebus` |
| onprem | `py_serverless_kit.runtime.onprem_host.submit_queue` | `put_object` / `delete_object` | `submit_topic` |
| S3-compatible (alibaba OSS, tencent COS, huawei OBS, oracle, ibm COS, DO Spaces, R2) | – | `boto3` with endpoint + HMAC keys | – |

For storage, **py-storebridge** gives one API across all of these — the natural pairing for portable pskit handlers.

Databases/caches: `psycopg[binary]`, `pymysql`, `pymongo`, `redis`, `pymemcache`, `smtplib`.

## Five steps for any resource

1. Add the client library to `requirements.txt` (and install locally).
2. Declare under `resources:`; for queues/buckets/topics add a trigger with `resource:`.
3. Read with `resource("key")`; secrets via `<name>_env` keys → `ctx.env[...]`.
4. Test locally: `pskit invoke -f worker -t queue -d '[...]'` or `pskit dev`.
5. Deploy and test: `pskit resources send KEY -d '{...}'`, `pskit resources check KEY`, `pskit logs -f`.

`pskit resources usage KEY` (studio: **Resources → How to use**) prints the full generated guide for your resource.

## On premises

- Built-in file broker for queues/topics/storage; or drivers: Redis (`options.url`), SQS, S3/MinIO/any S3-compatible, Azure Queue/Blob, Pub/Sub/GCS, MNS/OSS, Cloudflare Queues/R2, COS, OBS, Oracle, IBM COS, Spaces, directory, or a **custom class** (`claim`, `settle` required; `depth`, `recover` optional).
- One-click Docker services (studio **Resources → On-prem services → + Create service**, or `pskit resources onprem create KEY --service redis --attach worker --save`): Redis, Valkey, Kafka, NATS, RabbitMQ, ElasticMQ, MinIO, PostgreSQL, MySQL, MariaDB, MongoDB, Memcached, Mailpit, OpenBao, S3/Secrets Manager/SES compatible servers, LocalStack, and more (44 total).
- From other programs: `from py_serverless_kit.client import PskitClient` (or HTTP `/__pskit/queues/<name>` with `x-pskit-token`).

## Secrets

- Connection values and `${env:...}` values land in Terraform state (and on-prem `pskit_host.json`). Pass the **name** of a secret variable instead of the secret.
- Clouds: `environment: {NAME: ${env:LOCAL_NAME}}` or the cloud secret store; read `ctx.env[...]` or `get_secret()`.
- On-prem: `provider.options.env_file` (default `.env.onprem`), shipped as `host.env`.
- Local: `.env` loaded by `pskit invoke` / `pskit dev`.

## Troubleshooting

| Message | Fix |
|---|---|
| `pskit does not create database resources; set external: true` | database/cache/custom are always external |
| `function 'x' uses unknown resource 'y'` | add it under `resources:` |
| `... has a queue event on resource 'y', which is a storage` | resource type must match event |
| `managed queue resources that no event uses are not supported on X` | create outside, `external: true` + `name` |
| `resource 'x' is not bound to this function` (runtime) | add the function to `functions:` of the resource, or remove the list |
