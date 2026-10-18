# claude-code-system

Skills and agents for Claude Code that encode how a web team works with AI: from refining a
ticket to shipping, observing and operating the change in production.

They are **company-agnostic**. Organization-specific facts (repo names, clusters, namespaces, tracker
project, design-system links, which app uses which secret store) are left as placeholders or
"Fill this in for your organization" tables. Fill them in your own fork, or better, keep them in your
team wiki and let the skills read them from there.

## Placeholders

| Placeholder | Meaning |
|---|---|
| `<org>` | Your GitHub / tracker organization |
| `<app>` | An application / repository; `api`, `web`, `worker` are example apps |
| `<reference-app>` | The repo your team treats as the model for conventions |
| `PROJ`, `PROJ-123` | Your tracker project and ticket keys (`#123` on GitHub) |
| `<cluster>` | Your Kubernetes cluster / kube-context |
| `<app>-<env>` | Namespace convention (`staging`, `production`) |

## Skills

**Ticket lifecycle**

| Skill | What it covers |
|---|---|
| `/ticket-lifecycle` | The overall model: the ticket is the state, four checkpoints, who can pass each one |
| `/task-tracker` | Project, ticket hierarchy, epics, board states, discarding a ticket — for Jira, GitHub Projects, Asana or any tracker |
| `/refinement-context` | Where to find a ticket's context and how to write it down |
| `/acceptance-criteria` | Writing testable acceptance criteria and output artifacts |
| `/definition-of-ready` | Pass/fail check before a ticket moves to Refined |
| `/gate-policies` | Checkpoint policies as ticket labels (feature flag, QA, design, rollout) |

**Building and shipping**

| Skill | What it covers |
|---|---|
| `/gitflow` | Short-lived branches, conventional commits, squash merge, one subtask = one PR |
| `/pull-request` | Opening a PR with the team template and the ticket key linked |
| `/ci-cd` | What runs on the PR and how each app deploys |
| `/feature-flags` | Shipping unfinished work safely behind a flag |
| `/rollout-plan` | Sequencing breaking or destructive changes |
| `/app-design` | Screen designs with design systems and per-repo canvases |

**Configuration and operations**

| Skill | What it covers |
|---|---|
| `/secret-or-setting` | Decide whether a config value is a secret, a setting or a runtime switch |
| `/credentials` | Read and write secrets per store (sops, Rails credentials, GitHub secrets) |
| `/observability` | JSON logs in Loki, OpenTelemetry traces in Tempo, log contracts |
| `/grafana-dashboards` | Which panels to build from an app's log events, and their LogQL |
| `/kubectl` | Inspect and operate an app's namespace safely (staging default, production confirmed) |
| `/test-apps` | Smoke-test a UI or API against staging or local |

**Meta**

| Skill | What it covers |
|---|---|
| `/skill-design` | How to write or review a skill in this repo |

See [`docs/skill-design.md`](docs/skill-design.md) for how the skills fit together.

## Agents

| Agent | What it does |
|---|---|
| `architect` | Senior architect and reviewer: designs implementations and stops technical debt from spreading |

## Install

Copy (or symlink) `skills/*` into `~/.claude/skills/` and `agents/*` into `~/.claude/agents/`, or
into a project's `.claude/` directory to scope them to one repo.
