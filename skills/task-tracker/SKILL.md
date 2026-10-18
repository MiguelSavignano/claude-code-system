---
name: task-tracker
description: How work is created and tracked in your team's task tracker (Jira, GitHub Projects, Asana, Linear or any other) — the tracker-neutral model (one active project, the hierarchy Epic → Story/Task/Bug/Investigation → Subtask, an Epic is a goal and not a folder, Epic naming and the domain tag, sprints, the ticket states and their transitions — Refining, Refined, Validating, Discard —, the ticket fields for criteria, artifacts and design, blocking links, how to discard a ticket even when something is already installed) and how each concept maps to each tool, with one reference per tool. Use when creating or reorganizing tickets, deciding whether something is an Epic, reading or moving a ticket, or when someone asks "what states does the board have" or "which state do I move this to". Invoke as /task-tracker.
---

# Task tracker: where and how work is created

The other skills speak of **the tracker**, **a ticket** and **its key**. This skill is the only
place that says how that maps to your tool. Fill this in for your organization:

| | Value |
|---|---|
| Tool | e.g. Jira · GitHub Projects · Asana · Linear |
| Where | e.g. `https://<org>.atlassian.net`, `github.com/orgs/<org>/projects/<n>`, Asana project URL |
| Active project | `PROJ` (archived projects: read as history, never create or move work there) |
| Ticket key format | e.g. `PROJ-123` (Jira, Linear) · `#123` or `<org>/<repo>#123` (GitHub) · Asana task URL or custom ID |
| How to operate it | The MCP or CLI below; record any IDs it needs (cloudId, project number, workspace gid) |

How each concept is done in each tool — **read only the one your team uses**:
[Jira](references/jira.md) · [GitHub Projects](references/github-projects.md) ·
[Asana](references/asana.md). Another tool: copy one of them and fill in the same sections.

> **All new work goes in one project.** The backlog exists to **prioritize and review**; every
> unnecessary Epic makes it worse.

## Ticket types

| Level | Type | For |
|---|---|---|
| 1 | **Epic** | A goal, functional area or project spanning several sprints |
| 0 | **Story** | A user-facing requirement or capability |
| 0 | **Task** | Technical or implementation work (enablers included) |
| 0 | **Bug** | Fixing incorrect behavior that already exists |
| 0 | **Investigation** | A question that has to be decided, with no code to merge |
| -1 | **Subtask** | The steps a Story/Task is split into (one per PR) |

If the tool has no native type, the tool reference says how it is represented (issue type,
label, custom field). Verify against your project before creating.

## Epic: a goal, not a folder

Create an Epic only for a project, a management goal or an area with several Stories/Tasks
lasting several sprints. **Not** for a technical task, a bug, an endpoint, a migration, a UI
change or a one-off request.

> "Is this something the team would want to see as a main goal when reviewing the next
> sprints?" If not → Story/Task. **When in doubt, do not create an Epic.**

**Name:** a business capability or outcome, short and general; never a user story.
Good: "[OPS] Order status and fulfillment reporting". Bad: "The user wants to unlock the invoice".
**Domain tag** at the start, exactly one per Epic (fill in: `[SRE]` platform, `[<TAG>]` …). If
the work spans domains, it is several Epics or the scope is too broad. **Do not invent domains**: ask.

## Procedure for creating work

1. **Understand the request** without inventing scope: requirement, technical work, bug or question.
2. **Find the Epic** among the open ones. If none fits, create the ticket **without an Epic**
   (cheaper than an extra Epic).
3. **Choose the type** and create **the smallest ticket** that represents the request.
4. Criteria and output artifacts → `/acceptance-criteria`; where they are stored → [references/fields.md](references/fields.md).

**Sprints:** do not assign sprint or priority unless asked: the team sets them. A ticket can
start refining outside the sprint and continue inside it.

## Ticket states

A single workflow for all types. Use your tool's exact state names, verbatim, in moves and searches.

| State | Means | Exits via |
|---|---|---|
| `To Do` | Just created, not refined | triage → `Refining` |
| `Refining` | Criteria and output artifacts are being written | **ready to start** (the owner moves it; `/definition-of-ready`) → `Refined` |
| `Refined` | Spec approved; ready to plan | plan ready → `In Progress` |
| `In Progress` | Being built, one lane per subtask | **deployed** where it is validated → `Validating` |
| `Validating` | Installed where it is validated (staging, or production behind a flag) | **validated and delivered** → `Done` · rework → `In Progress` · new criterion → `Refining` |
| `Done` | Reached the customer | — |
| `Discard` | Not going to be done, and nothing is left installed | — ([references/discard.md](references/discard.md)) |

Most trackers do not stop a ticket from jumping states: `/definition-of-ready` checks that, not
the tool. **Do not move a ticket on your own** unless asked, and never from `Refining` to `Refined`.
**Beware of automations that close tickets** (e.g. a PR merge closing an issue): merged is not
delivered. Each tool reference says how to avoid it.

## Blocking links

"A blocks B" means B cannot start until A is Done. Check the direction before creating one:
backlogs often contain Done tickets "blocked by" tickets that have not started — a link created
in the wrong direction. Point them out; do not fix them without asking.
