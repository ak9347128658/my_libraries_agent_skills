# Workflows (state machines)

Defined once in `pskit.yml`; run natively as AWS Step Functions, Google Cloud Workflows or (opt-in `engine: native`) Azure Logic Apps; elsewhere by the embedded engine in a generated `<workflow>-orchestrator` function; locally with `pskit workflow run`.

```yaml
functions:
  validate: {handler: app.orders.validate}
  charge:   {handler: app.orders.charge}
  notify:   {handler: app.orders.notify}

workflows:
  order-pipeline:
    description: Validate, charge and notify
    start_at: Validate
    states:
      Validate:
        type: task
        function: validate
        retry: [{errors: [TimeoutError], max_attempts: 3, interval_seconds: 2, backoff_rate: 2}]
        catch: [{errors: [ValueError], next: Rejected}]
        next: Route
      Route:
        type: choice
        choices:
          - {variable: "$.amount", gt: 1000, next: ManualReview}
        default: Charge
      ManualReview: {type: wait, seconds: 3600, next: Charge}
      Charge:       {type: task, function: charge, result_path: "$.payment", next: Notify}
      Notify:       {type: task, function: notify, result_path: null, end: true}
      Rejected:     {type: fail, error: OrderRejected, cause: validation failed}
```

Task handlers get `event.type == "workflow"` and the state input in `event.data`; their return value is the result. Raise exceptions to trigger `retry`/`catch` — error names are **Python exception class names**.

```python
def validate(event, ctx):
    order = dict(event.data)
    if order.get("amount", 0) <= 0:
        raise ValueError("amount must be positive")
    return {**order, "valid": True}
```

## Workflow keys

| Key | Default | Notes |
|---|---|---|
| `start_at`, `states` | – | required |
| `description` | `""` | |
| `engine` | `auto` | native on aws/gcp, embedded elsewhere (also Azure); `native` errors where unsupported; `embedded` forces engine |
| `timeout` | – | run timeout |
| `enabled` | true | |
| `options.type` | STANDARD | AWS: STANDARD / EXPRESS |
| `options.http` | true | embedded: `false` removes the public `POST /workflows/<name>` route |
| `options.start_on_entry` | true | on-prem: first task function running alone starts the workflow |
| `options.sync` | true | Azure native: `false` returns 202 immediately |

## States

| type | Fields |
|---|---|
| `task` | `function`, `next`/`end`, `input_path`, `result_path`, `retry`, `catch`, `timeout_seconds` |
| `choice` | `choices` (rules + `next`), `default` (no match & no default → `States.NoChoiceMatched`) |
| `wait` | `seconds` or `seconds_path` |
| `pass` | `result`, `result_path` |
| `parallel` | `branches: [{start_at, states}]` → list result |
| `map` | `items_path`, `iterator: {start_at, states}`, `max_concurrency` → list result |
| `succeed` | – |
| `fail` | `error`, `cause` |

`task/wait/pass/parallel/map` without `next` = end. Any state may have `comment`.

## Data flow

JSON paths: `$`, `$.a.b`, `$.items[0]`, `$['key']`. `input_path` selects the task input (default `$`). `result_path`: `$` replaces data (default), `$.x` stores under `x`, `null` discards the result.

## Choice rules

```yaml
choices:
  - {variable: "$.status", equals: "paid", next: Ship}
  - {variable: "$.amount", gte: 100, next: Review}
  - and:
      - {variable: "$.country", in: [IN, US]}
      - not: {variable: "$.vip", equals: true}
    next: Standard
  - or: [{variable: "$.email", is_present: false}, {variable: "$.email", is_null: true}]
    next: AskEmail
```

Operators: `equals`, `not_equals`, `gt`, `gte`, `lt`, `lte`, `in`, `contains`, `matches` (glob `*`/`?`), `is_null`, `is_present`, `and`, `or`, `not`. Missing/wrong-type variable → false.

## Retry / catch

```yaml
retry:
  - errors: [TimeoutError, ConnectionError]    # "*" or States.ALL = any
    max_attempts: 3
    interval_seconds: 1
    backoff_rate: 2.0
    max_delay_seconds: 60
catch:
  - errors: ["*"]
    next: HandleError
    result_path: "$.error"                    # stores {"error", "cause"}
```

## Engines and limits

| Provider | Engine |
|---|---|
| aws | Step Functions (ASL) |
| gcp | Cloud Workflows (only the first retry rule; retry/catch on tasks only) |
| azure `engine: native` | Logic App (first retry rule only, 120 s task calls, sequential `map`, limited `matches`, ~2 min sync response) |
| others / `embedded` | `<workflow>-orchestrator` function, tasks in-process |

Embedded on clouds: **whole run must fit one invocation** (AWS/Huawei/Tencent 900 s, Azure 600 s, IBM 120 s) and returns synchronously `{"execution_id", "status", "output", "duration_ms"}` or 500 FAILED. On-prem: tracked by the host, `POST /workflows/<name>` → 202 + `execution_id` (`?wait=true` to block), list/follow/cancel.

Not supported: `Parameters`, `ResultSelector`, `OutputPath`, intrinsic functions, `.sync`/callback integrations, timestamp waits — do that shaping in a task function.

## Commands

```bash
pskit workflow run order-pipeline -i '{"amount": 50}' --history     # local, waits skipped (--sleep to honour)
pskit workflow show order-pipeline --format asl
pskit workflow start order-pipeline -i @events/order.json           # deployed
pskit workflow status <id> --watch --history
```

Studio: **Workflows → + Create workflow** → add states from the palette (new state chains after the selected one) → click a state to edit function/next/Retry/Catch → **Save** → **Test** on the card runs it with a live trace.
