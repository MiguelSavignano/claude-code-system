# Deprecated

These agents and skills were removed from the repo in `95068d0` and are kept here for reference only.
They are not maintained.

## Why

The repo now focuses on **harness engineering**: skills that shape how Claude Code works inside a
team's process (ticket lifecycle, checkpoints, CI/CD, configuration, observability, operations),
rather than one-off task automations or stack-specific coding rules.

The items below don't fit that focus. They automate a single task, enforce conventions for one
codebase or language, or hard-code paths and commands from a specific project.

## Contents

**Agents**

| Agent | What it did |
|---|---|
| `agents/ci-failure-resolver.md` | Diagnose and fix a failed CI/CD pipeline or GitHub Actions workflow |
| `agents/npm-vulnerabilities-patcher.md` | Audit npm dependencies, fix critical/high vulnerabilities and open a PR |
| `agents/ruby-gem-scaffolder.md` | Scaffold a new Ruby gem project with CI/CD configuration |

**Skills**

| Skill | What it did |
|---|---|
| `skills/backend-architecture` | Enforce Controller/Service/Lib/Guard layering in Node.js, Python and Rails |
| `skills/http-outbound-tracing` | Enforce the `[HTTP_OUTBOUND]` log format on external HTTP calls |
| `skills/system-design` | Enforce architecture and visual consistency from `SYSTEM_DESIGN.md` |
| `skills/web-design-guidelines` | Review UI code against Web Interface Guidelines |
| `skills/commands/controller.md` | Apply the decorator pattern to Django controllers in one project |
| `skills/commands/debug-server.md` | Start one project's uvicorn server and fix errors until it boots |

Outbound HTTP logging now lives in `skills/observability` (see `references/http-outbound.md`).
