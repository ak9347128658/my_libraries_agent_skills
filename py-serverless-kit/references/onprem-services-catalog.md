# One-click on-prem services catalog (pskit 3.2.3)

Generated from `pskit resources onprem catalog --json`. Use the **id** with `--service`. In studio, these are the cards under **Resources → On-prem services → + Create service** (the card label is in the **Studio card** column).

## Create one

```bash
pskit resources onprem catalog                                         # this list
pskit resources onprem servers add db-box --ssh ubuntu@10.0.1.12 --pem ~/.ssh/app.pem --sudo
pskit resources onprem create orders-db --service postgres --server db-box --save     # on a saved server
pskit resources onprem create jobs --service redis --role queue --attach worker --save # queue + trigger on 'worker'
pskit resources onprem create cache --service redis --role cache --local --save       # on this machine
pskit resources onprem create docker --service docker --server app-1                  # install Docker on a server
pskit resources onprem usage orders-db --reveal                                       # address, credentials, client code
pskit resources onprem status|logs|backup|tunnel|console orders-db
pskit resources onprem remove orders-db --delete-data --save
```

Useful flags: `--role` (queue, stream, cache, database, storage, custom); `--server NAME` or `--ssh user@host --pem FILE --sudo` or `--local`; `--port`; `--user`, `--password-env VAR` (if no password is given, a strong one is generated and kept out of pskit.yml); `--gui` (also start the web UI); `-a/--attach FN` (add a queue or storage trigger, repeatable); `--save` (write the resource, `provider.options.env_file` and triggers to pskit.yml). Passwords go to the env file (default `.env.onprem`), never to `pskit.yml`.

In handlers, read the service with `resource("<key>")`. For example, a database gives `host`, `port`, `database`, `user` and `password_env`, so the password is `ctx.env[r["password_env"]]`. The exact keys for your resource are in `pskit resources usage <key>`, or **How to use** on its row in studio.

## Caches and queues

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `redis` | Redis | queue, stream, cache | 6379 | Redis Commander | Queue (redis driver: leased lists), stream (redis_stream driver: consumer group, several hosts) or a cache for handlers. |
| `valkey` | Valkey | queue, stream, cache | 6379 | Redis Commander | Open-source Redis fork; same redis and redis_stream drivers. |
| `dragonfly` | Dragonfly (Redis-compatible) | queue, stream, cache | 6379 | Redis Commander | A faster Redis-compatible store: the same redis / redis_stream drivers and clients. Password only (user default). |
| `keydb` | KeyDB (Redis-compatible) | queue, stream, cache | 6379 | Redis Commander | Multithreaded Redis fork: the same redis / redis_stream drivers and clients. |
| `memcached` | Memcached | cache | 11211 |  | In-memory cache; nothing to back up. |

## Queues and streams

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `rabbitmq` | RabbitMQ | queue | 5672 | Management UI | AMQP with manual acks through a small driver class pskit adds to your app (needs pika). Management UI on port 15672. Backups hold definitions (queues, users), not messages. |
| `elasticmq` | ElasticMQ (SQS-compatible) | queue | 9324 | Queue browser | Amazon SQS API, used through the sqs driver. Keeps messages in memory: a restart empties it, so use it for development and tests. UI on port 9325. |
| `kafka` | Kafka (Redpanda, single node) | queue | 9092 | Kafka UI | Kafka API (Redpanda, no ZooKeeper). Consumed through a driver class pskit adds to your app (needs kafka-python). No authentication: keep the port on a private network. |
| `nats` | NATS JetStream | queue | 4222 | Monitoring (JSON) | A JetStream stream with a durable pull consumer, through a driver class pskit adds to your app (needs nats-py). Monitoring on port 8222. |
| `apache-kafka` | Apache Kafka (KRaft, single node) | queue | 9092 | Kafka UI | The Apache Kafka distribution (no ZooKeeper), consumed through the Kafka driver class pskit adds (needs kafka-python). No authentication: keep the port on a private network. |
| `lavinmq` | LavinMQ (AMQP 0.9.1) | queue | 5672 | Management UI | A light, fast AMQP broker that speaks RabbitMQ's protocol: the RabbitMQ driver class and clients work unchanged. Management UI on port 15672. |
| `mosquitto` | Eclipse Mosquitto (MQTT broker) | custom | 1883 |  | MQTT for IoT devices and sensors. Anonymous access: keep it on a private network. |

## Databases

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `postgres` | PostgreSQL | database | 5432 | Adminer (database UI) |  |
| `mysql` | MySQL | database | 3306 | Adminer (database UI) |  |
| `mariadb` | MariaDB | database | 3306 | Adminer (database UI) |  |
| `mongodb` | MongoDB | database | 27017 | Mongo Express |  |
| `qdrant` | Qdrant (vector database) | database | 6333 | Dashboard | Vector similarity search for embeddings (RAG, recommendations). REST on the port, gRPC on 6334; the API key is the password. |
| `clickhouse` | ClickHouse (analytics database) | database | 8123 | SQL playground | Column-store SQL for analytics and logs. HTTP on the port (SQL playground at /play), native protocol on 9000. |
| `neo4j` | Neo4j (graph database) | database | 7687 | Neo4j Browser | Graph database queried with Cypher over Bolt (the port); Neo4j Browser on 7474. The user is always neo4j. |

## Object storage

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `minio` | MinIO (S3-compatible) | storage | 9000 | Console | Buckets watched through the minio driver. Console on port 9001. |
| `azurite` | Azurite (Azure Blob + Queue emulator) | queue, storage | 10000 |  | Azure Storage blobs (port 10000) and queues (10001) with your own account name and key: the azure_blob / azure_queue drivers and the Azure SDKs work unchanged. Browse it with Azure Storage Explorer. |
| `fake-gcs` | fake-gcs-server (Cloud Storage emulator) | storage | 4443 |  | Google Cloud Storage JSON API: the gcs driver (emulator mode) and the google-cloud-storage library work with its endpoint. No authentication. For development and tests. |
| `rustfs` | RustFS (S3-compatible) | storage | 9000 | Console | An S3-compatible object store with a web console (port 9001), an alternative to MinIO: the s3 driver, boto3 and the AWS CLI work with its endpoint. |
| `s3` | Amazon S3 compatible (Python) | storage | 9000 | S3 console | pskit's own S3 server (Python, code in aws/s3): the S3 API with SigV4 authentication (user = access key ID, password = secret access key). boto3, the AWS SDKs, the AWS CLI and the s3 driver work with its endpoint. Ver… |
| `s3-node` | Amazon S3 compatible (Node.js) | storage | 9000 | S3 console | pskit's own S3 server (Node.js, code in aws/s3): the S3 API with SigV4 authentication (user = access key ID, password = secret access key). boto3, the AWS SDKs, the AWS CLI and the s3 driver work with its endpoint. Ve… |

## AWS emulators

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `localstack` | LocalStack (SQS, S3, Secrets Manager) | queue, storage, secrets | 4566 | Status (JSON) | Amazon SQS, S3, Secrets Manager and SSM Parameter Store on one port: the sqs / s3 drivers, get_secret(), boto3 and the AWS CLI work with endpoint_url. Credentials are not checked (test / test). For development and tests. |
| `localemu` | LocalEmu (AWS emulator: SQS, S3, Secrets Manager, ...) | queue, storage, secrets | 4566 | Dashboard | An open-source AWS emulator (Apache-2.0 fork of LocalStack, 130+ services) on one port: the sqs / s3 drivers, get_secret(), boto3, the AWS CLI and Terraform work with endpoint_url. Credentials are not checked (test / … |

## Secrets

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `openbao` | OpenBao (Vault-compatible secrets) | secrets | 8200 |  | A secret manager with the HashiCorp Vault API (KV v2 at secret/): get_secret() / put_secret(), hvac, node-vault and the vault / bao CLI. The password is the access token. OpenBao's release images ship without its web … |
| `secretsmanager` | AWS Secrets Manager compatible (Python) | secrets | 4584 | Secrets console | pskit's own Secrets Manager server (Python, code in aws/secretsmanager): the Secrets Manager API with SigV4 authentication (user = access key ID, password = secret access key). boto3, the AWS SDKs, the AWS CLI and get… |
| `secretsmanager-node` | AWS Secrets Manager compatible (Node.js) | secrets | 4584 | Secrets console | pskit's own Secrets Manager server (Node.js, code in aws/secretsmanager): the Secrets Manager API with SigV4 authentication (user = access key ID, password = secret access key). boto3, the AWS SDKs, the AWS CLI and ge… |
| `infisical` | Infisical (secrets manager with web UI) | secrets | 8080 | Infisical | Open-source secrets manager (projects, environments, versions, audit log, access control) with its own PostgreSQL and Redis next to it. Open the web UI and create the admin account, then a project and a machine identi… |

## Tools

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `mailpit` | Mailpit (SMTP) | custom | 1025 | Mailbox | An SMTP server that keeps every mail for inspection (web UI on port 8025). For development and tests. |
| `portainer` | Portainer (Docker UI) | custom | 9000 | Portainer | Manage the server's containers, images, volumes and networks in the browser. Create the admin account within 5 minutes of the first start (otherwise restart it). |
| `gitea` | Gitea (Git hosting) | custom | 3000 | Gitea | Self-hosted Git with pull requests, issues, packages and Actions. Finish the install page on the first visit; the first account is the admin. |
| `git` | Git | runtime |  |  | git from the distribution's packages. |
| `tools` | Basic tools | runtime |  |  | curl, wget, unzip, jq, htop, tmux and netcat for working on the server. |

## Mail

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `smtp` | SMTP server (receive mail, Node.js) | smtp | 25 |  | Receives mail for your domains and runs the functions attached to it with the mail: from, to, cc, subject, text, html, headers and attachments (also in the Amazon SES event shape). Never relays. After creating it, How… |
| `ses` | Amazon SES compatible (Python) | custom | 4579 | SES console | pskit's own Amazon SES server (Python, code in aws/ses): the SES v1 and v2 APIs with SigV4 authentication (user = access key ID, password = secret access key) and an SES-style SMTP endpoint (port 2587). boto3 (ses / s… |
| `ses-cpp` | Amazon SES compatible (C++) | custom | 4579 | SES console | pskit's own Amazon SES server (C++, code in aws/ses): the SES v1 and v2 APIs with SigV4 authentication (user = access key ID, password = secret access key) and an SES-style SMTP endpoint (port 2587). boto3 (ses / sesv… |

## Monitoring

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `grafana` | Grafana (dashboards) | custom | 3000 | Grafana | Dashboards and alerts over Prometheus, Loki, PostgreSQL, ClickHouse and 100+ data sources. |
| `prometheus` | Prometheus (metrics) | custom | 9090 | Prometheus UI | Metrics database and query UI. Scrapes itself by default; add targets in /etc/prometheus/prometheus.yml (Console: sh). No login: keep it on a private network. |
| `uptime-kuma` | Uptime Kuma (uptime monitor) | custom | 3001 | Uptime Kuma | Checks HTTP, TCP, DNS, Docker containers ... every minute and alerts by mail, Slack, Telegram. Create the admin account on the first visit. |
| `jaeger` | Jaeger (tracing, OpenTelemetry) | custom | 16686 | Jaeger UI | Distributed tracing: send OpenTelemetry traces to OTLP gRPC :4317 or HTTP :4318 and search them in the UI. Keeps traces in memory. No login: keep it on a private network. |

## Identity

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `keycloak` | Keycloak (login, SSO, OAuth2 / OIDC) | custom | 8080 | Admin console | Identity provider: users, groups, social login, MFA and OpenID Connect / OAuth2 tokens for your APIs. Runs in dev mode (H2 database in the volume, HTTP); put it behind HTTPS for production. |

## Search

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `elasticsearch` | Elasticsearch (search and analytics) | database | 9200 | Kibana | Full-text search, logs and analytics with the Elasticsearch REST API on the port. User elastic, the password is the service password. Web UI: Kibana (Also start its web UI). Needs about 2 GB of memory (1 GB heap). |
| `meilisearch` | Meilisearch (search engine) | custom | 7700 | Search preview | Typo-tolerant full-text search with a REST API and a search preview UI (the master key is the password). |
| `typesense` | Typesense (search engine) | custom | 8108 |  | Fast typo-tolerant search with a REST API (the API key is the password). |

## Automation

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `n8n` | n8n (workflow automation) | custom | 5678 | n8n | Visual workflow automation with 400+ integrations; call your pskit functions with its HTTP node or the Lambda-compatible invoke API. Create the owner account on the first visit. |

## Web

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `nginx-proxy-manager` | Nginx Proxy Manager (reverse proxy, Let's Encrypt) | custom | 81 | Admin UI | Point domains at your functions (port 8080) and services with free HTTPS certificates, from a web UI. First login admin@example.com / changeme (you are asked to change it). Needs ports 80 and 443 free. |

## Languages and Docker

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `docker` | Docker | runtime |  |  | Installs and starts Docker on an SSH server (needed by every service above, and by method: docker). |
| `python` | Python 3 | runtime |  |  | system: the distribution's python3 with venv and pip. 3.X: that exact CPython (standalone build through uv, any distro) as /usr/local/bin/python3.X, so the host runs with python: system and runtime python3.X. |

## Languages

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `nodejs` | Node.js | runtime |  |  | Node.js and npm from NodeSource (apt / dnf; Alpine: its own version). Needed by Node.js functions with node: system, PM2 and npm-based tools. |
| `java` | Java (OpenJDK) | runtime |  |  | Headless OpenJDK runtime from the distribution's packages. |
| `go` | Go | runtime |  |  | The official Go toolchain in /usr/local/go (linked into /usr/local/bin). |
| `cpp` | C / C++ (GCC) | runtime |  |  | GCC (gcc, g++), make, CMake, Ninja, pkg-config, gdb and the OpenSSL headers from the distribution's packages. A version adds that GCC as g++-N (apt, or gcc-toolset-N on RHEL / Alma / Rocky). Builds the C++ SES server … |

## Web servers

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `nginx` | nginx | runtime |  |  | nginx from the distribution's packages, enabled and started. Put a reverse proxy to the pskit host (port 8080) in /etc/nginx/conf.d/. |
| `caddy` | Caddy | runtime |  |  | The caddy binary in /usr/local/bin (automatic HTTPS). Try it with: caddy reverse-proxy --from :80 --to 127.0.0.1:8080. |
| `certbot` | Certbot (Let's Encrypt) | runtime |  |  | certbot with its nginx plugin where the distribution has it (dnf needs EPEL). |

## Process managers

| id | Studio card | use as (`--role`) | port | web UI | notes |
|---|---|---|---|---|---|
| `pm2` | PM2 | runtime |  |  | PM2 (npm install -g pm2) to keep Node.js apps running; install Node.js first. |
