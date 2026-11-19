# Testing pskit handlers

Locally, handlers run through the **same dispatcher** as in the cloud (routing, CORS, error mapping, BatchResult); only the cloud adapter is skipped. No credentials or Terraform needed.

## pytest with LocalRunner (pattern from examples/14-testing)

```python
# tests/conftest.py
from pathlib import Path
import pytest
from py_serverless_kit.core.project import Project
from py_serverless_kit.local import LocalRunner

ROOT = Path(__file__).resolve().parents[1]

@pytest.fixture(scope="session")
def project() -> Project:
    return Project.load(cwd=ROOT)            # also: config_path=, provider=, stage=

@pytest.fixture
def runner(project) -> LocalRunner:
    return LocalRunner(project, env_file=None)   # default env_file=".env"
```

```python
# tests/test_handlers.py
import json
from py_serverless_kit.local import build_event, http_event

def body(resp):
    status, headers, raw = resp.render()
    return status, json.loads(raw) if raw else None

def test_get_item(runner):
    resp = runner.invoke("get-item", http_event("GET", "/items/1"))
    assert resp.status == 200

def test_create_validation(runner):
    status, data = body(runner.invoke("create-item", http_event("POST", "/items", {"amount": -1})))
    assert status == 400

def test_invalid_json(runner):
    event = http_event("POST", "/price", "{not json", headers={"content-type": "application/json"})
    assert body(runner.invoke("price", event))[0] == 400

def test_queue_partial_failure(runner):
    result = runner.invoke("ingest", build_event("queue", [{"id": 1}, "garbage"]))
    assert result.failed == ["local-1"]           # record ids are local-0, local-1, ...

def test_schedule(runner):
    assert runner.invoke("nightly", build_event("schedule", {"report": "daily"})) is not None
```

| Helper | Purpose |
|---|---|
| `Project.load(config_path=None, provider=..., stage=..., cwd=...)` | load + validate |
| `LocalRunner(project, env_file=".env")` | in-process runner (sets `PSKIT_PROVIDER=local`, resources env, `.env`) |
| `runner.invoke(function, event)` | HTTP → `Response` (`.status`, `.render()` → `(status, headers, bytes)`); batch → `BatchResult` (`.failed`); else the return value |
| `runner.run_workflow(name, data, sleep=False)` | `{"output", "history"}`; raises `WorkflowFailed` (`.error`) |
| `http_event(method, path, body=None, headers=None, query=None)` | dict/list body JSON-encoded; `?query` parsed |
| `build_event(kind, data, **http)` | `queue`/`topic` (list → records), `storage` (`{"bucket","key","size","action"}`), `schedule`, `invoke`, `workflow`, `http` |

## Workflow tests

```python
import pytest
from py_serverless_kit.runtime.workflow import WorkflowEngine, WorkflowFailed
from py_serverless_kit.workflows import definition_dict

def test_end_to_end(runner):
    out = runner.run_workflow("scoring", {"customer": "ada", "vip": True})
    assert out["output"]["score"] == 90

def test_declines(runner):
    with pytest.raises(WorkflowFailed) as exc:
        runner.run_workflow("scoring", {"customer": "bob"})
    assert exc.value.error == "LowScore"

def test_routing_with_stubs(project):
    def fake_task(function, data):
        return {**data, "score": 85} if function == "score" else data
    engine = WorkflowEngine(definition_dict(project.config.workflows["scoring"]), fake_task)
    assert engine.run({"x": 1})["score"] == 85
```

## Direct unit tests

Handlers are plain functions; for pure logic call helpers directly and keep handlers thin. Use LocalRunner whenever routing, error mapping or batch behaviour matters.

## CLI and dev server

```bash
pskit invoke -f hello --path "/hello?name=Ada" -H "Authorization: Bearer x"
pskit invoke -f worker -t queue -d @events/batch.json
pskit workflow run order-pipeline -i @events/order.json --history
pskit dev                                   # http://127.0.0.1:3000, hot reload
curl -X POST 'localhost:3000/__pskit/invoke/process-jobs?type=queue' -d '[{"job":1}]'
curl -X POST localhost:3000/__pskit/workflows/order-pipeline -d '{"amount":50}'
```

Studio: **Test → pick function → choose event → ▶ Run locally**.

## CI (examples/14-testing/ci.github-workflow.yml)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12", cache: pip}
      - run: pip install py-serverless-kit pytest -r requirements.txt
      - run: python -m pytest -q
  validate:
    runs-on: ubuntu-latest
    strategy: {matrix: {provider: [aws, gcp, azure]}}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}
      - run: pip install py-serverless-kit
      - run: pskit validate --provider ${{ matrix.provider }}
      - run: pskit validate --provider ${{ matrix.provider }} --stage prod
      - run: pskit package --provider ${{ matrix.provider }} --skip-deps   # main.tf.json for review
```

Deploy-per-stage workflow: `examples/15-ci-cd-deploy/deploy.github-workflow.yml` (needs remote `provider.state` and cloud credentials as CI secrets).
