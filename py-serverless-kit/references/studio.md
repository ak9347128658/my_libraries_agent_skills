# pskit studio: navigation hub

Use this file to answer **"how do I … in studio?"** questions:

1. Find the task in **Quick answers** below. That gives the click path.
2. For the full steps, open the linked page under [`studio/`](studio/). Each page is the official studio guide for one page or panel, with screenshots removed.
3. If the task isn't listed, search the **Complete section index** at the bottom. It lists every section of every studio page.

Answer in the studio's own words. Give the click path in the form **Sidebar page → section → button**, using the exact labels in **bold**. Then name the `pskit.yml` change the click makes, and give the equivalent CLI command when one exists.

## Start and open studio

```bash
pip install py-serverless-kit
cd my-service          # a folder with pskit.yml, or an empty folder for a new project
pskit studio --open    # flags: -d DIR, --host 127.0.0.1, --port 8770, --no-auth, --vscode-image IMG, --ailogs
```

Open the exact printed link `http://127.0.0.1:8770/#token=...`. A new token is made at every start; without it the page says **This studio session needs its link**. Press **Ctrl+C** in the terminal to stop studio. Details: [studio/start.md](studio/start.md).

## Screen map

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ λ service │ stage [dev] │ ↶ ↷ │ N errors · N warnings │ Validate │ Save │ ⇅ Settings │ ◐ │  ← top bar
├──────────────────────┬────────────────────────────────────────────────────────┤
│ ◎ Overview & provider│                                                        │
│ λ Functions      (N) │                 page content                           │
│ ⇉ Workflows      (N) │                                                        │
│ ▤ Resources      (N) │                                                        │
│   Scaling cluster    │  (on-prem pages show a warning on other providers)     │
│   Load balancers     │                                                        │
│   Network map        │                                                        │
│ {} Code              │                                                        │
│ ▶ Test               │                                                        │
│ ▲ Deploy             │                                                        │
│ ☰ Logs               │                                                        │
│ ≡ YAML               │                                                        │
│   Users & audit      │                                              (Hana) ◯  │  ← Ctrl+I
└──────────────────────┴────────────────────────────────────────────────────────┘
Hidden page: Setup & credentials → add #/setup to the studio URL (or the link on Logs)
Empty folder → "Create a pskit project" page instead of the sidebar
```

| Sidebar | Guide page | Edits in pskit.yml | CLI twin |
|---|---|---|---|
| New project (empty folder) | [new-project.md](studio/new-project.md) | whole new file | `pskit init` |
| Overview & provider | [overview.md](studio/overview.md) | `service`, `provider`, `providers` | edit yaml |
| Setup & credentials (`#/setup`) | [setup.md](studio/setup.md) | `provider.options.deploy` (on-prem) | `pskit doctor`, `pskit settings` |
| Functions | [functions.md](studio/functions.md), [trigger-forms.md](studio/trigger-forms.md) | `functions` | `pskit function split` |
| Workflows | [workflows.md](studio/workflows.md) | `workflows` | `pskit workflow ...` |
| Resources (+ servers, services) | [resources.md](studio/resources.md), [servers.md](studio/servers.md), [onprem-services.md](studio/onprem-services.md), [replication.md](studio/replication.md), [monitoring.md](studio/monitoring.md) | `resources` | `pskit resources ...` |
| Scaling cluster | [cluster.md](studio/cluster.md) | `provider.options.cluster` | `pskit cluster ...` |
| Load balancers | [load-balancers.md](studio/load-balancers.md) | `provider.options.balancer` | `pskit lb ...` |
| Network map | [network.md](studio/network.md) | – | `pskit firewall` |
| Code | [code.md](studio/code.md) | handler files | your editor |
| Test | [test.md](studio/test.md) | `.pskit/studio/test-events.json` | `pskit invoke`, `pskit trigger` |
| Deploy (+ Placement, Firewall) | [deploy.md](studio/deploy.md), [placement.md](studio/placement.md), [firewall.md](studio/firewall.md) | `providers.<cloud>` disables, `provider.options.placement` | `pskit validate/package/plan/deploy/info/remove` |
| Logs | [logs.md](studio/logs.md) | – | `pskit logs` |
| YAML | [yaml.md](studio/yaml.md) | anything | edit yaml |
| Users & audit | [accounts-and-team.md](studio/accounts-and-team.md) | `.pskit-vault/` | `pskit vault`, `pskit login` |
| Hana (avatar, Ctrl+I) | [hana.md](studio/hana.md) | after you approve | – |
| A full app, click by click | [end-to-end.md](studio/end-to-end.md) | – | – |

## How drafts and saving work (applies to every page)

- Each edit changes an **in-memory draft**. The toast says "(not saved yet)". Nothing is written to disk until you press **Save** or **Ctrl+S**.
- About 0.6 s after you stop typing, studio validates the draft for the provider and stage in the top bar. **N errors · N warnings** opens **Problems**. Click a problem to jump to its field.
- **Save** writes `pskit.yml` and any queued handler stubs, after copying the old file to `.pskit/studio/backups/` (the last 20 are kept).
- **Save** also drops comments, re-orders keys, keeps `${env:..}`/`${self:..}`/`${opt:..}` as written, edits only the main file when `extends:` is used, and lets you save an invalid config.
- If the file changed on disk, Save offers **Cancel**, **Reload (discard my changes)** or **Overwrite**. Deploy and Test read the file from disk, so they ask **Save first?** first.
- Undo/redo keeps 100 steps across pages: **Ctrl+Z** / **Ctrl+Shift+Z**.

## Quick answers: "How do I … in studio?"

### Project and provider
| Task | Click path | Writes / CLI |
|---|---|---|
| Create a new project | Empty folder → `pskit studio --open` → **Create a pskit project**: **Service name**, provider card, **Language**, **Template** (HTTP API / Events / Workflow / Everything) → **Create project** | `pskit init NAME -p P -t http\|events\|workflow\|full -l python` · [new-project](studio/new-project.md) |
| Rename the service | **Overview & provider → Service → Service name** → Save | `service:` (renaming creates new cloud resources) |
| Switch cloud (e.g. AWS → Azure) | **Overview & provider → Provider** → click the card → **Switch & disable on <Provider>** or **Switch only (fix later)** → Save | `provider.name`. The old region and options move to `providers.<old>` · [overview](studio/overview.md#switch-the-provider-in-studio) |
| Set region/runtime/memory/timeout for all functions | **Overview & provider → Defaults for every function** | `provider.region/runtime/memory/timeout/architecture/log_retention_days` |
| Add env vars for all functions | **Overview & provider → Environment variables** (use `${env:NAME}` for secrets) | `provider.environment` |
| Set the Azure subscription, GCP project and other provider options | **Overview & provider → <Provider> options** | `provider.options.*` |
| Change the stage | top bar **stage** field | `--stage` for validate, deploy, test and logs |
| Fix the "providers.<cloud> overrides a setting" box | **Overview & provider** → yellow box → **Use the values shown here** → Save | moves the values into `provider:` |
| Change packaging (source, excludes, vendor) | **YAML → Edit raw YAML** (there's no form for this) | `package.*` |

### Credentials and machine setup
| Task | Click path | Writes / CLI |
|---|---|---|
| Check that the tools and credentials are ready | URL `#/setup` → pick **Provider** and **Operating system** → read **Checks** → **How to fix** → **Check again** | `pskit doctor -p P` · [setup](studio/setup.md) |
| Log in to Azure / GCP / AWS SSO | `#/setup` → **Sign in (az login)** / **Sign in (gcloud ADC)** / **Sign in (aws sso)** | runs the cloud CLI login |
| Save an AWS access key | `#/setup` (provider AWS) → **AWS credentials (aws configure)** → **Profile**, keys → **Save profile** | `~/.aws/credentials` |
| Set `ARM_SUBSCRIPTION_ID` or another provider variable | `#/setup` → **Environment variables** → value → **Set for this session** | lasts only until studio stops; nothing is written to disk |
| Create `.venv` and install requirements | `#/setup` → **Project environment** → **Python** → **Create .venv and install** | `python -m venv .venv` + pip |
| Move servers, services and SSH logins to another PC | top bar **⇅ Settings** (or `#/setup` → **Move settings to another machine**) → passphrase → **Export settings**. On the other PC: pick the file → **Preview** → **Import into this project** | `pskit settings export/import` |

### Functions and triggers
| Task | Click path | Writes / CLI |
|---|---|---|
| Create a function | **Functions → + Create function** → **Basics → Triggers → Environment & permissions → Review** → **Create function** → **Save** | `functions.<name>` + a handler stub · [functions](studio/functions.md#create-a-function-with-the-studio-wizard) |
| Give a function its own dependencies | Wizard **Basics → Dependencies: Own folder…** (on by default). For an existing function: **Functions** row → **Own folder** → **Move it** | `functions/<name>/handler.py` + `requirements.txt` · `pskit function split FN` |
| Add an HTTP route | Wizard **Triggers → HTTP endpoint** → **Route** (method + `/path/{id}`), **Auth**, **CORS** | `- http: ...` · [trigger-forms](studio/trigger-forms.md#add-an-http-endpoint-trigger-in-studio) |
| Add a cron/schedule | Wizard **Triggers → Schedule** → **Every…** / **At a specific time** / **Cron expression**, then **Timezone**, **Input** | `- schedule: ...` |
| Add a queue / storage / topic trigger | Wizard **Triggers → Queue/Storage/Topic → Source**: **Create new** (+ **Create & select**) / **Project resource** / **Pick from <Provider>** / **Type a name** | `- queue: {name\|resource ...}` · [trigger-forms](studio/trigger-forms.md#add-a-queue-storage-or-topic-trigger-in-studio) |
| Use an existing cloud queue or bucket | Trigger form → **Source: Pick from <Provider>** → filter → click the item (keep **Also add to Resources (external)** on) | `existing: true` or an external resource |
| Set env vars, log level or IAM for one function | Function row → wizard step **Environment & permissions** | `functions.<fn>.environment`, `PSKIT_LOG_LEVEL`, `iam` |
| Add a trigger to an existing function | **Functions** → click the row (opens **Edit <name>**) → click the **Triggers** step name → **Add a trigger** → card → fill the form → **Apply changes** → **Save** | adds an item to `functions.<fn>.events`. The CLI has no add-trigger command: edit `pskit.yml`, then run `pskit validate` |
| Disable a function on one cloud only | **Functions** row → **Deploy** dropdown → **◐ Disabled on <Provider>** | `providers.<p>.functions.<fn>.enabled: false` |
| Disable a function on a cloud that is **not** the active one (e.g. GCP while AWS is active) | *<Provider>* in the studio dropdowns always means the provider picked on **Overview & provider**. Either switch to that provider, disable the function, **Save** and switch back (the override stays under `providers.gcp`), or use **YAML → Edit raw YAML** to add `providers: {gcp: {functions: {<fn>: {enabled: false}}}}` → **Apply to draft** → **Save** | check with `pskit validate -p gcp` and `pskit validate -p aws` |
| Disable one trigger on one cloud | Wizard **Triggers** → the trigger's **Disable on <Provider>** | `providers.<p>.functions.<fn>.events` |
| Duplicate or delete a function | **Functions** row → **Duplicate** / **Delete** (HTTP routes are not copied; the code is not deleted) | |
| Edit handler code | **Code** (or **Code** on a function row) → edit → **Ctrl+S**. For full VS Code: switch to **Full VS Code** → **Pull & start** (needs Docker) | [code](studio/code.md) |

### Workflows
| Task | Click path | Writes / CLI |
|---|---|---|
| Create a workflow | **Workflows → + Create workflow** → **Name** → **Create** → add states from the palette → **Save** | `workflows.<name>` · [workflows](studio/workflows.md) |
| Add a task, choice, wait, parallel or map step | Select a state → palette button (the new state chains after it) → inspector: **Function** / **Rules** / **Seconds** / **Branches** / **Items path** | |
| Add branching | choice state → **+ Add rule** → **if / if not / all of / any of** → condition → **then go to** → **Otherwise** | `choices`, `default` |
| Add retries or error handling | task state → **Retry (N) → + retry rule**, **Catch (N) → + catch rule** | `retry`, `catch` |
| Choose the engine or a timeout | editor **Settings** → **Engine** (auto/native/embedded), **Timeout (s)** | `engine`, `timeout` |
| Run or test a workflow | **Test** on the workflow card or in the editor → **This machine** / **Deployed** | `pskit workflow run NAME --history` |

### Resources, services and servers
| Task | Click path | Writes / CLI |
|---|---|---|
| Add a bucket, queue, topic or database | **Resources → + Add resource** → **Key**, **Type**, **Mode** (Managed / External), **Native name**, **Connection**, **Bound functions** → **Add** → **Save** | `resources.<key>` · [resources](studio/resources.md#add-a-resource-in-studio) |
| See how to use a resource in code | **Resources** row → **How to use** (**Copy guide**) | `pskit resources usage KEY` |
| Send a test message or upload a test object | **Resources → Test a resource** → **Test connection** → edit the JSON → **Send a message** / **Upload an object** / **Publish a message** | `pskit resources send KEY -d ...` |
| Delete cloud buckets left after a remove | **Resources → In the cloud** → **Load** → tick → **Delete selected** → type `delete` | `pskit resources delete ADDR` |
| Pick an on-prem driver (Redis, S3, SQS…) | Resource drawer → **On-premises driver** | `resources.<k>.options` |
| Start Postgres, Redis, MinIO… with one click (on-prem) | **Resources → On-prem services → + Create service** → pick a card (e.g. **PostgreSQL** under **Databases**) → **Resource key**, **Where** (this machine / **One SSH server** → **Server**), **Security** (leave **Password** empty to generate one) → **Create** → **Save** | `pskit resources onprem create orders-db --service postgres --server db-box --save`. All 59 service ids are in [onprem-services-catalog.md](onprem-services-catalog.md) · [onprem-services](studio/onprem-services.md) |
| Choose which functions get a database or cache | **Resources** → click the resource row → **Bound functions** (none ticked = all functions) → **Apply** → **Save**. Handlers read it with `resource("key")`; the password is `ctx.env[r["password_env"]]` | `resources.<key>.functions: [...]` |
| Let **cloud** functions (Azure, AWS…) use a database on my own server | **Resources → + Add resource** → **Type** `database` → **Mode: External / existing** → **Connection**: `host`, `port`, `database`, `user`, `password_env` = the *name* of a variable → **Bound functions** → **Add**. Put the secret itself in **Overview & provider → Environment variables** as `${env:DB_PASSWORD}`. The server's port must be reachable from the cloud | `resources.<key>: {type: database, external: true, connection: {...}}` |
| Save an SSH server | **Resources → On-prem servers → + Add server** → **Name**, **Host**, **User**, **Log in with** → **Save and test** | `pskit resources onprem servers add` · [servers](studio/servers.md) |
| Open a shell or terminal on a server or service | server/service row → **Shell** / **Terminal** / **Console** | `pskit resources onprem shell/console/terminal` |
| Back up a service | service row → **Backup** | `pskit resources onprem backup KEY` |
| Create an SMTP server for an e-mail trigger | **Resources → + SMTP server** → follow the **DNS guide** | [trigger-forms](studio/trigger-forms.md#add-an-e-mail-smtp-trigger-in-studio) |
| Replicated database | [replication.md](studio/replication.md) | |
| Server CPU / memory / disk and alerts | [monitoring.md](studio/monitoring.md) | |

### Test, deploy and logs
| Task | Click path | Writes / CLI |
|---|---|---|
| Run a function locally | **Test** → function → **1. Choose the event** → **2. Edit the event** → **▶ Run locally** | `pskit invoke -f FN ...` · [test](studio/test.md) |
| Save a test payload | **Test** → **Saved events** → name → **Save event** | `.pskit/studio/test-events.json` |
| Send a real event to the deployed function | **Test** → choose the event → **☁ Send to <provider>/<stage>** → **Send** (turn on *Show the function's logs*) | `pskit trigger -f FN -e 0 -d ...` |
| Validate / package / plan | **Deploy** → **Validate** / **Package** / **Plan** | `pskit validate/package/plan -p P -s S` |
| Deploy | set the stage in the top bar → **Deploy** → **Deploy** (a `prod*` stage asks you to type `provider/stage`) | `pskit deploy -p P -s S` · [deploy](studio/deploy.md) |
| See URLs after a deploy | **Deploy → Info** | `pskit info` |
| Fix "won't run on <Provider>" | **Deploy** → **N items won't run on <Provider>** → **Disable … on <Provider>** / **Disable all of them** → Save | `providers.<p>` overrides |
| Remove a stage but keep its data | **Deploy** → turn the remove switch off (**keeps buckets, queues and topics**) → **Remove** → type `provider/stage` | `pskit remove --keep-resources` |
| Deploy on-prem to a server or cluster | **Deploy → Run on** → **Machine** / **Scaling cluster** | `pskit deploy --server NAME` / `--cluster NAME` |
| Split functions across servers | **Deploy → Where functions run**, or wizard **Basics → Runs on** | `provider.options.placement` · [placement](studio/placement.md) |
| Which ports to open | **Deploy → Firewall**, **Network map** → click a machine | `pskit firewall` · [firewall](studio/firewall.md) |
| Watch logs | **Logs** → filter **Level** / **Function** / **Source** → **❚❚ Pause** | `pskit logs -f` · [logs](studio/logs.md) |
| Show logs for one function only | URL `#/logs/<function>`, or wizard step 3 **Live logs** | `pskit logs -f --function FN` |
| Delete old logs | **Logs → Stored logs** → tick → **Delete selected** | `pskit logs --delete-all` |
| Change a function's log level | wizard step **Environment & permissions → Logging → Log level** | `PSKIT_LOG_LEVEL` |

### YAML, on-prem scaling, team and Hana
| Task | Click path | Writes / CLI |
|---|---|---|
| See what Save will write | **YAML → Diff vs disk** | [yaml](studio/yaml.md) |
| Paste or edit raw YAML | **YAML → Edit raw YAML** → **Apply to draft** → **Save** | |
| Create a scaling cluster | **Scaling cluster → + New cluster** → **Cluster name**, **Orchestrator** (Swarm / k3s / Nomad), **Master server** → **Create** → **+ Join server** → turn on **Deploy this project's functions to …** → Save → Deploy | `pskit cluster create` · [cluster](studio/cluster.md) |
| Add a load balancer | **Load balancers → + Create load balancer** → **Name**, **Server**, algorithm → **Create** | `pskit lb` · [load-balancers](studio/load-balancers.md) |
| Set up a team | first start → **A team** → create the admin → save the **Master key** and **Recovery code** → **Users & audit → Users & permissions → Add a user** (preset viewer / developer / operator) | `pskit vault init`, `pskit vault users add` · [accounts-and-team](studio/accounts-and-team.md) |
| See who changed what | **Users & audit → Audit log** | `pskit vault audit` |
| Ask the assistant | avatar at bottom right or **Ctrl+I** → first time: **Where should Hana run?** → **Install & start** (Docker, ~4 GB) → ask → approve the plan | [hana](studio/hana.md) |

## Complete section index

Every section of every studio guide page, generated from the page headings. Use it when a task isn't in the quick answers above.

### [pskit studio guide](studio/start.md)
- [Install and start pskit studio](studio/start.md#install-and-start-pskit-studio)
- [pskit studio command-line flags](studio/start.md#pskit-studio-command-line-flags)
- [The studio session token and the link](studio/start.md#the-studio-session-token-and-the-link)
- [The studio sidebar map](studio/start.md#the-studio-sidebar-map)
- [The studio top bar](studio/start.md#the-studio-top-bar)
- [How studio drafts, validates and saves pskit.yml](studio/start.md#how-studio-drafts-validates-and-saves-pskityml)
- [Studio keyboard shortcuts](studio/start.md#studio-keyboard-shortcuts)
- [Where pskit studio stores its files](studio/start.md#where-pskit-studio-stores-its-files)

### [New project page in studio](studio/new-project.md)
- [Create a new project in studio](studio/new-project.md#create-a-new-project-in-studio)
- [Create the same project from the terminal](studio/new-project.md#create-the-same-project-from-the-terminal)

### [Overview & provider page in studio](studio/overview.md)
- [Rename the service in studio](studio/overview.md#rename-the-service-in-studio)
- [Switch the provider in studio](studio/overview.md#switch-the-provider-in-studio)
- [Fix "providers.<cloud> overrides a setting" in studio](studio/overview.md#fix-providerscloud-overrides-a-setting-in-studio)
- [Set defaults for every function in studio](studio/overview.md#set-defaults-for-every-function-in-studio)
- [Set environment variables and tags for every function](studio/overview.md#set-environment-variables-and-tags-for-every-function)
- [Set provider options in studio](studio/overview.md#set-provider-options-in-studio)
- [Change packaging and on-prem host settings in YAML](studio/overview.md#change-packaging-and-on-prem-host-settings-in-yaml)

### [Setup & credentials page in studio](studio/setup.md)
- [Check tools and credentials in studio](studio/setup.md#check-tools-and-credentials-in-studio)
- [Save an AWS access key as a profile in studio](studio/setup.md#save-an-aws-access-key-as-a-profile-in-studio)
- [Set provider environment variables for the studio session](studio/setup.md#set-provider-environment-variables-for-the-studio-session)
- [Create the project's .venv or run npm install in studio](studio/setup.md#create-the-projects-venv-or-run-npm-install-in-studio)
- [Edit the on-prem SSH connection on the Setup page](studio/setup.md#edit-the-on-prem-ssh-connection-on-the-setup-page)
- [Move studio settings to another machine](studio/setup.md#move-studio-settings-to-another-machine)

### [Functions page in studio](studio/functions.md)
- [Read the functions table in studio](studio/functions.md#read-the-functions-table-in-studio)
- [Enable or disable a function on one provider](studio/functions.md#enable-or-disable-a-function-on-one-provider)
- [Duplicate or delete a function in studio](studio/functions.md#duplicate-or-delete-a-function-in-studio)
- [Give a function its own folder in studio](studio/functions.md#give-a-function-its-own-folder-in-studio)
- [Create a function with the studio wizard](studio/functions.md#create-a-function-with-the-studio-wizard)
- [Wizard step 1: Basics](studio/functions.md#wizard-step-1-basics)
- [Wizard step 2: Triggers](studio/functions.md#wizard-step-2-triggers)
- [Wizard step 3: Environment & permissions](studio/functions.md#wizard-step-3-environment--permissions)
- [Wizard step 4: Review](studio/functions.md#wizard-step-4-review)

### [Trigger forms in studio](studio/trigger-forms.md)
- [Read the provider limitations box on a trigger](studio/trigger-forms.md#read-the-provider-limitations-box-on-a-trigger)
- [Add an HTTP endpoint trigger in studio](studio/trigger-forms.md#add-an-http-endpoint-trigger-in-studio)
- [Add a schedule trigger in studio](studio/trigger-forms.md#add-a-schedule-trigger-in-studio)
- [Add a queue, storage or topic trigger in studio](studio/trigger-forms.md#add-a-queue-storage-or-topic-trigger-in-studio)
- [Pick an existing queue, bucket or topic from your cloud account](studio/trigger-forms.md#pick-an-existing-queue-bucket-or-topic-from-your-cloud-account)
- [Add an e-mail (SMTP) trigger in studio](studio/trigger-forms.md#add-an-e-mail-smtp-trigger-in-studio)

### [Workflows page in studio](studio/workflows.md)
- [Read the workflow list in studio](studio/workflows.md#read-the-workflow-list-in-studio)
- [Create a workflow in studio](studio/workflows.md#create-a-workflow-in-studio)
- [Add and connect states in the workflow builder](studio/workflows.md#add-and-connect-states-in-the-workflow-builder)
- [Edit a state in the workflow inspector](studio/workflows.md#edit-a-state-in-the-workflow-inspector)
- [Write choice rules in the workflow builder](studio/workflows.md#write-choice-rules-in-the-workflow-builder)
- [Add retry and catch to a workflow state](studio/workflows.md#add-retry-and-catch-to-a-workflow-state)
- [Set the workflow engine and timeout in studio](studio/workflows.md#set-the-workflow-engine-and-timeout-in-studio)
- [Run a workflow from studio or the terminal](studio/workflows.md#run-a-workflow-from-studio-or-the-terminal)

### [Resources page in studio](studio/resources.md)
- [Read the resources table in studio](studio/resources.md#read-the-resources-table-in-studio)
- [Add a resource in studio](studio/resources.md#add-a-resource-in-studio)
- [Pick an on-prem driver for a resource](studio/resources.md#pick-an-on-prem-driver-for-a-resource)
- [Delete a resource in studio](studio/resources.md#delete-a-resource-in-studio)
- [Open the How to use guide of a resource](studio/resources.md#open-the-how-to-use-guide-of-a-resource)
- [Test a resource in studio](studio/resources.md#test-a-resource-in-studio)
- [Browse the data of an on-prem queue, topic or bucket](studio/resources.md#browse-the-data-of-an-on-prem-queue-topic-or-bucket)
- [Delete deployed or kept cloud resources in studio](studio/resources.md#delete-deployed-or-kept-cloud-resources-in-studio)
- [Read the resource guides inside studio](studio/resources.md#read-the-resource-guides-inside-studio)

### [Servers panel in studio](studio/servers.md)
- [Add a server in studio](studio/servers.md#add-a-server-in-studio)
- [Test and watch a saved server in studio](studio/servers.md#test-and-watch-a-saved-server-in-studio)
- [Open a shell or a terminal on a saved server](studio/servers.md#open-a-shell-or-a-terminal-on-a-saved-server)
- [Install software or create a service on a saved server](studio/servers.md#install-software-or-create-a-service-on-a-saved-server)
- [See where a server is used, edit it or forget it](studio/servers.md#see-where-a-server-is-used-edit-it-or-forget-it)

### [On-prem services panel in studio](studio/onprem-services.md)
- [Read the on-prem services table in studio](studio/onprem-services.md#read-the-on-prem-services-table-in-studio)
- [Create an on-prem service in studio](studio/onprem-services.md#create-an-on-prem-service-in-studio)
- [Install software on a server from studio](studio/onprem-services.md#install-software-on-a-server-from-studio)
- [Open a service's details, web UI and logs](studio/onprem-services.md#open-a-services-details-web-ui-and-logs)
- [Use How to use for an on-prem service](studio/onprem-services.md#use-how-to-use-for-an-on-prem-service)
- [Run commands in a service: Console and Terminal](studio/onprem-services.md#run-commands-in-a-service-console-and-terminal)
- [Browse buckets and secrets with pskit's own browser](studio/onprem-services.md#browse-buckets-and-secrets-with-pskits-own-browser)
- [Back up or remove an on-prem service](studio/onprem-services.md#back-up-or-remove-an-on-prem-service)

### [Database replication in studio](studio/replication.md)
- [Create a replicated database in studio](studio/replication.md#create-a-replicated-database-in-studio)
- [Add or remove a read replica in studio](studio/replication.md#add-or-remove-a-read-replica-in-studio)

### [Monitoring and alerts in studio](studio/monitoring.md)
- [Watch a server's CPU, memory and disk in studio](studio/monitoring.md#watch-a-servers-cpu-memory-and-disk-in-studio)
- [Watch a service's containers in studio](studio/monitoring.md#watch-a-services-containers-in-studio)
- [Turn on alerts for an on-prem service](studio/monitoring.md#turn-on-alerts-for-an-on-prem-service)

### [Scaling cluster page in studio](studio/cluster.md)
- [Create a scaling cluster in studio](studio/cluster.md#create-a-scaling-cluster-in-studio)
- [Deploy the project's functions to a cluster in studio](studio/cluster.md#deploy-the-projects-functions-to-a-cluster-in-studio)
- [Join, remove and watch cluster servers in studio](studio/cluster.md#join-remove-and-watch-cluster-servers-in-studio)
- [Put a load balancer in front of a cluster](studio/cluster.md#put-a-load-balancer-in-front-of-a-cluster)
- [Scale replicas and watch requests and logs on a cluster](studio/cluster.md#scale-replicas-and-watch-requests-and-logs-on-a-cluster)
- [Alert when every cluster server is full](studio/cluster.md#alert-when-every-cluster-server-is-full)

### [Load balancers page in studio](studio/load-balancers.md)
- [Create a shared load balancer in studio](studio/load-balancers.md#create-a-shared-load-balancer-in-studio)
- [Put this project behind a load balancer in studio](studio/load-balancers.md#put-this-project-behind-a-load-balancer-in-studio)
- [Manage a load balancer in studio](studio/load-balancers.md#manage-a-load-balancer-in-studio)
- [Attach clusters and read routes on a load balancer](studio/load-balancers.md#attach-clusters-and-read-routes-on-a-load-balancer)

### [Network map page in studio](studio/network.md)
- [Read the network map in studio](studio/network.md#read-the-network-map-in-studio)
- [See the ports one machine must allow](studio/network.md#see-the-ports-one-machine-must-allow)

### [Firewall panel in studio](studio/firewall.md)
- [Read the firewall panel in studio](studio/firewall.md#read-the-firewall-panel-in-studio)
- [Get the firewall rules from the terminal](studio/firewall.md#get-the-firewall-rules-from-the-terminal)

### [Placement (Where functions run) in studio](studio/placement.md)
- [Split functions across servers in studio](studio/placement.md#split-functions-across-servers-in-studio)
- [Pick where one function runs (Runs on)](studio/placement.md#pick-where-one-function-runs-runs-on)
- [See where groups were deployed](studio/placement.md#see-where-groups-were-deployed)

### [Code page in studio](studio/code.md)
- [Edit handler code in the built-in editor](studio/code.md#edit-handler-code-in-the-built-in-editor)
- [Open the project in full VS Code from studio](studio/code.md#open-the-project-in-full-vs-code-from-studio)

### [Test page in studio](studio/test.md)
- [Run a function locally from studio](studio/test.md#run-a-function-locally-from-studio)
- [Send a real event to the deployed function](studio/test.md#send-a-real-event-to-the-deployed-function)
- [Save named test events in studio](studio/test.md#save-named-test-events-in-studio)
- [Read How to test this for a trigger](studio/test.md#read-how-to-test-this-for-a-trigger)

### [Deploy page in studio](studio/deploy.md)
- [Deploy a project from studio](studio/deploy.md#deploy-a-project-from-studio)
- [Validate, package, plan and show info from studio](studio/deploy.md#validate-package-plan-and-show-info-from-studio)
- [Remove a deployed stage from studio](studio/deploy.md#remove-a-deployed-stage-from-studio)
- [Fix triggers and workflows that won't run on the target provider](studio/deploy.md#fix-triggers-and-workflows-that-wont-run-on-the-target-provider)
- [Choose the machine or cluster for an on-prem deploy](studio/deploy.md#choose-the-machine-or-cluster-for-an-on-prem-deploy)
- [Read the deploy steps of an on-prem deploy](studio/deploy.md#read-the-deploy-steps-of-an-on-prem-deploy)
- [Invoke a function or run a workflow locally](studio/deploy.md#invoke-a-function-or-run-a-workflow-locally)
- [Review recent deploy jobs](studio/deploy.md#review-recent-deploy-jobs)

### [Logs page in studio](studio/logs.md)
- [Watch live logs in studio](studio/logs.md#watch-live-logs-in-studio)
- [Delete stored log streams in studio](studio/logs.md#delete-stored-log-streams-in-studio)
- [Set a function's log level in studio](studio/logs.md#set-a-functions-log-level-in-studio)

### [YAML page in studio](studio/yaml.md)
- [Preview and diff pskit.yml in studio](studio/yaml.md#preview-and-diff-pskityml-in-studio)
- [Edit raw YAML and apply it to the draft](studio/yaml.md#edit-raw-yaml-and-apply-it-to-the-draft)

### [Accounts and team setup in studio](studio/accounts-and-team.md)
- [The first start: just me or a team](studio/accounts-and-team.md#the-first-start-just-me-or-a-team)
  - [Change your mind later](studio/accounts-and-team.md#change-your-mind-later)
- [Set up a team](studio/accounts-and-team.md#set-up-a-team)
  - [What the admin does](studio/accounts-and-team.md#what-the-admin-does)
  - [What each teammate does](studio/accounts-and-team.md#what-each-teammate-does)
- [The admin account](studio/accounts-and-team.md#the-admin-account)
- [The master key, the password and the recovery code](studio/accounts-and-team.md#the-master-key-the-password-and-the-recovery-code)
- [How your data is protected](studio/accounts-and-team.md#how-your-data-is-protected)
- [How user permissions work](studio/accounts-and-team.md#how-user-permissions-work)
  - [Add a user](studio/accounts-and-team.md#add-a-user)
  - [The presets](studio/accounts-and-team.md#the-presets)
  - [What read means for secrets](studio/accounts-and-team.md#what-read-means-for-secrets)
  - [Saving pskit.yml](studio/accounts-and-team.md#saving-pskityml)
  - [Permissions to give with care](studio/accounts-and-team.md#permissions-to-give-with-care)
  - [Change or remove access](studio/accounts-and-team.md#change-or-remove-access)
  - [What permissions cannot stop](studio/accounts-and-team.md#what-permissions-cannot-stop)

### [Hana panel in studio](studio/hana.md)
- [Open the Hana chat panel](studio/hana.md#open-the-hana-chat-panel)
- [Install Hana's model from studio](studio/hana.md#install-hanas-model-from-studio)
- [Approve or reject Hana's changes](studio/hana.md#approve-or-reject-hanas-changes)
- [Use studio cards in the Hana chat](studio/hana.md#use-studio-cards-in-the-hana-chat)
- [Explain pages, the project or validation with Hana](studio/hana.md#explain-pages-the-project-or-validation-with-hana)
- [Manage Hana chats](studio/hana.md#manage-hana-chats)
- [Log every Hana step in the terminal](studio/hana.md#log-every-hana-step-in-the-terminal)

### [Build a full application in studio, click by click (orders service)](studio/end-to-end.md)
- [Start pskit studio in an empty folder](studio/end-to-end.md#start-pskit-studio-in-an-empty-folder)
- [Create the project in the Create a pskit project screen](studio/end-to-end.md#create-the-project-in-the-create-a-pskit-project-screen)
- [How saving works in studio](studio/end-to-end.md#how-saving-works-in-studio)
- [Check the machine on Setup & credentials](studio/end-to-end.md#check-the-machine-on-setup--credentials)
- [Set the service defaults on Overview & provider](studio/end-to-end.md#set-the-service-defaults-on-overview--provider)
- [Remove the sample function on Functions](studio/end-to-end.md#remove-the-sample-function-on-functions)
- [Add the resources on the Resources page](studio/end-to-end.md#add-the-resources-on-the-resources-page)
- [Create the HTTP functions with the Create function wizard](studio/end-to-end.md#create-the-http-functions-with-the-create-function-wizard)
- [Create the queue consumer and the storage processor](studio/end-to-end.md#create-the-queue-consumer-and-the-storage-processor)
- [Create the nightly schedule and the topic subscribers](studio/end-to-end.md#create-the-nightly-schedule-and-the-topic-subscribers)
- [Create the workflow step functions](studio/end-to-end.md#create-the-workflow-step-functions)
- [Build the fulfil-order workflow on the Workflows page](studio/end-to-end.md#build-the-fulfil-order-workflow-on-the-workflows-page)
- [Write the handler code on the Code page](studio/end-to-end.md#write-the-handler-code-on-the-code-page)
- [Add the stage override and hooks on the YAML page](studio/end-to-end.md#add-the-stage-override-and-hooks-on-the-yaml-page)
- [Test the functions locally on the Test page](studio/end-to-end.md#test-the-functions-locally-on-the-test-page)
- [Deploy the dev stage on the Deploy page](studio/end-to-end.md#deploy-the-dev-stage-on-the-deploy-page)
- [Send real events and read logs](studio/end-to-end.md#send-real-events-and-read-logs)
- [Deploy the prod stage and remove a stage](studio/end-to-end.md#deploy-the-prod-stage-and-remove-a-stage)
- [Ask Hana to do the same steps](studio/end-to-end.md#ask-hana-to-do-the-same-steps)

