# Library agent skills

Two independent Agent Skills (the `SKILL.md` + `references/` format used by Claude Code, the Claude Agent SDK and the Claude API skills feature):

| Skill | Library | Version covered |
|---|---|---|
| [`py-serverless-kit/`](py-serverless-kit/SKILL.md) | [py-serverless-kit](https://github.com/kavion-ai/py-serverless-kit) (`pskit`) — one handler, 10 clouds + on-prem, studio web UI | 3.2.x |
| [`py-storebridge/`](py-storebridge/SKILL.md) | [py-storebridge](https://pypi.org/project/py-storebridge/) — one storage API over 11 providers, cross-provider transfer | 1.1.0 |

Each folder is self-contained; install either one alone or both.

## Install into an agent

**Claude Code — one project:** copy the folder(s) into the project's `.claude/skills/`:

```powershell
Copy-Item -Recurse py-serverless-kit  <project>\.claude\skills\
Copy-Item -Recurse py-storebridge     <project>\.claude\skills\
```

**Claude Code — every project:** copy them into `%USERPROFILE%\.claude\skills\` (`~/.claude/skills/` on macOS/Linux).

**Claude Agent SDK:** put them in `.claude/skills/` of the agent's working directory (or `~/.claude/skills/`) and enable skills in the SDK options (`setting_sources` including `"project"`/`"user"`, and the `Skill` tool allowed).

**Claude API (skills upload):** zip a single skill folder (the zip root must contain `SKILL.md`) and upload it as a custom skill.

The agent loads a skill automatically when the task matches its `description` (the frontmatter at the top of each `SKILL.md`); the files under `references/` are read only when needed.

## Layout

```
py-serverless-kit/
  SKILL.md                                   overview, studio vs CLI, handler + pskit.yml essentials, rules
  references/studio.md                       studio hub: screen map, "how do I" click paths, full section index
  references/studio/*.md                     25 studio guide pages (every page, panel, button), from the repo docs
  references/handler-api.md                  Event, Context, Response, errors, BatchResult, invoke, resource, secrets
  references/pskit-yml.md                    full config schema, triggers, variables, overrides
  references/cli.md                          all commands and flags
  references/workflows.md                    state machines
  references/resources.md                    resources, on-prem services, secrets
  references/testing.md                      LocalRunner, pytest, CI
  references/providers-and-troubleshooting.md
  references/migrate-from-aws-lambda.md
  references/examples.md                     the 21 official examples
py-storebridge/
  SKILL.md                                   install extras, core API, rules, transfer
  references/providers.md                    credentials for all 11 providers incl. on-prem modes
  references/api.md                          every method, model and error
  references/transfer-middleware-hooks.md    transfer, middleware, hooks, multipart, custom provider
```

## Keeping them current

When a library releases a new version, update the version line in its `SKILL.md` and re-check the reference tables (CLI flags: `pskit COMMAND --help`; storebridge: `py_storebridge/__init__.py` and `config.py`).
