# pskit CLI

`pskit --help` lists commands; `pskit COMMAND --help` shows every flag with examples. `py-serverless-kit` is an alias.

Common project flags: `-c/--config FILE`, `-p/--provider NAME`, `-s/--stage NAME`, `-r/--region NAME`, `-v/--verbose` (tracebacks). Errors print one `Error: …` line and exit 1.

## Getting started

| Command | Purpose |
|---|---|
| `pskit init [DIR] --name N -p PROVIDER -t http|events|workflow|full [-l python|javascript|typescript] [--typescript] [--runtime nodejs22] [--force]` | scaffold project |
| `pskit studio [-d DIR] [--host 127.0.0.1] [--port 8770] [--open] [--vscode-image IMG] [--ailogs]` | web UI; open the printed `#token=` URL |
| `pskit doctor [-p PROVIDER] [-c FILE] [--json]` | tools, CLI, credentials, required options. ✓ ok, ✗ missing (exit 1), ! recommended |

## Develop

| Command | Purpose |
|---|---|
| `pskit validate [-p P] [-s S]` | schema + provider limits + handler modules exist; exit 1 on errors |
| `pskit invoke -f FN [-t http|schedule|queue|storage|topic|invoke|workflow] [-d JSON|@file|text] [--method M] [--path '/x?y=1'] [-H 'Name: v'] [--env-file .env]` | run one handler locally; HTTP prints `{status, headers, body}`; exit 1 if it raises |
| `pskit dev [--host] [--port 3000] [--env-file .env]` | local HTTP server, hot reload; `POST /__pskit/invoke/<fn>?type=queue`, `POST /__pskit/workflows/<wf>`, dashboard at `/__pskit/` |
| `pskit function split FN` | move a function into `functions/<name>/` with its own requirements (Python) |

```bash
pskit invoke -f hello --path "/hello?name=Ada"
pskit invoke -f create-item -d '{"name": "lamp"}'
pskit invoke -f worker -t queue -d '[{"id": 1}, {"id": 2}]'
pskit invoke -f on-upload -t storage -d '{"bucket": "b", "key": "in/a.csv"}'
pskit invoke -f nightly                         # schedule input from config
curl -X POST 'localhost:3000/__pskit/invoke/worker?type=queue' -d '[{"job":1}]'
```

## Deploy

| Command | Purpose |
|---|---|
| `pskit package [--skip-deps] [--package-type zip|image|auto]` | build `.pskit/<p>-<s>/artifact.zip` + `main.tf.json`, size vs limits, `resources.json` |
| `pskit plan [--skip-deps] [--destroy] [--package-type ...]` | terraform plan |
| `pskit deploy [--skip-deps] [--confirm] [--upgrade] [--no-dashboard] [--all-or-nothing] [--package-type ...]` | package + apply; partial deploys list failed functions at the end (exit 1) |
| `pskit info [--json] [--refresh]` | functions, URLs, ids, console links |
| `pskit remove [-y] [--keep-resources]` | terraform destroy; keep buckets/queues/topics with `--keep-resources` |
| `pskit tf [-s S] -- ARGS` | any terraform command in the workspace (`state list`, `output -json`, `import ...`) |

## Operate

| Command | Purpose |
|---|---|
| `pskit trigger -f FN [-e N] [-d JSON] [--path P] [--direct] [--list] [--json]` | real event to deployed function |
| `pskit logs [-f] [-n 100] [--function F] [--level WARNING] [--source function|host|stdout|stderr|invocation] [-q TEXT] [--json] [--streams] [--delete ID] [--delete-all] [-y]` | on-prem buffer, AWS CloudWatch, GCP Logging, Azure Log Analytics |
| `pskit dashboard [-d .] [-o FILE] [--serve] [--port 8765] [--open]` | HTML dashboard of every deployment |
| `pskit dlq list|inspect FN ID|replay FN ID [--all] [--to Q]` | dead-letter queues |
| `pskit resources list|delete ADDR|send KEY -d ...|usage KEY|check KEY|data KEY|clear KEY` | managed/kept resources; usage guide per resource |
| `pskit resources onprem catalog|create KEY --service S|status|usage|backup|remove|servers ...` | one-click Docker services |
| `pskit providers [--json]` | capability matrix |

`pskit trigger` payloads: http = body; queue = message or list; storage = `{"key","content"}` (`"action":"deleted"` deletes); topic = `{"message","attributes"}`; schedule/invoke = `event.data`.

## Workflows

```bash
pskit workflow run NAME [-i JSON|@file] [--sleep] [--history]   # local; exit 2 on failure
pskit workflow show NAME [--format auto|asl|gcp|json]
pskit workflow start NAME [-i ...] [--wait] [--json]             # deployed
pskit workflow executions [NAME] [--status RUNNING|SUCCEEDED|FAILED|TIMED_OUT|CANCELLED|ABORTED]
pskit workflow status EXECUTION_ID [-w NAME] [--history] [--watch]
pskit workflow cancel EXECUTION_ID [-w NAME]
```

## On-prem, team, settings

| Command | Purpose |
|---|---|
| `pskit cluster create|join|status|scale|...` | Swarm / k3s / Nomad scaling |
| `pskit lb ...`, `pskit firewall` | load balancers, ports per server |
| `pskit client info|install-node|tunnel` | call the on-prem host from scripts |
| `pskit vault init`, `pskit login`, `logout`, `whoami`, `vault users add NAME --preset developer`, `vault audit` | team mode (encrypted `.pskit-vault/`); scripts use `PSKIT_MASTER_KEY`, `PSKIT_USER`, `PSKIT_PASSWORD` |
| `pskit settings export [-o FILE] [--with-config]` / `import FILE [--dry-run] [--replace]` | move `.pskit` settings between machines (passphrase-encrypted; `PSKIT_SETTINGS_PASSPHRASE`) |

## Environment variables

`PSKIT_TERRAFORM_BIN`, `PSKIT_LOG_FORMAT`, `PSKIT_LOG_LEVEL`/`LOG_LEVEL`, `PSKIT_DEBUG`, `PSKIT_SETTINGS_PASSPHRASE`, `PSKIT_STUDIO_VSCODE_IMAGE`, `PSKIT_HANA_AILOGS`.
