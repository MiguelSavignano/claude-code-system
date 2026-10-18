---
name: jira-task
description: How work is created and tracked in Jira — the single project where new work goes (`PROJ`), the hierarchy Epic → Story/Task/Bug/Investigation → Subtask, the rule that an Epic is a goal and not a folder, Epic naming and the domain tag prefix, sprints, and the ticket states with their transitions (Refining, Refined, Validating, Discard), the ticket fields (criteria, artifacts, design) and how to discard a ticket even when something is already installed. Use when creating or reorganizing issues, deciding whether something is an Epic, or when someone asks "what states does the board have" or "which state do I move this to". Invoke as /jira-task.
---

# Jira: where and how work is created

Site: `https://<org>.atlassian.net` — get the `cloudId` with `getAccessibleAtlassianResources`
and record it in your team wiki. Operate it with the Atlassian MCP (`searchJiraIssuesUsingJql`,
`getJiraIssue`, `createJiraIssue`, `editJiraIssue`, `getTransitionsForJiraIssue`).

> **All new work goes in one project (`PROJ`).** If older projects are archived, they are not
> deleted: their tickets and links remain for reference, but do not create or move anything there.
> If someone hands you a key from an archived project, read it as history and create the new work
> in `PROJ`. (Fill in for your organization: which project is active, which are archived.)

The backlog exists to **prioritize and review**. Every unnecessary Epic makes it worse.

## Issue types in PROJ

Verify against your project before creating: if something does not match, call
`getJiraProjectIssueTypesMetadata` first.

| Level | Type | For |
|---|---|---|
| 1 | **Epic** | A goal, functional area or project spanning several sprints |
| 0 | **Story** | A user-facing requirement or capability |
| 0 | **Task** | Technical or implementation work (enablers included) |
| 0 | **Bug** | Fixing incorrect behavior that already exists |
| 0 | **Investigation** | A question that has to be decided, with no code to merge (a custom type; name it as your project does) |
| -1 | **Subtask** | The steps a Story/Task is split into (one per PR) |

## Epic: a goal, not a folder

Create an Epic only for a project, a management goal or an area with several Stories/Tasks
lasting several sprints. **Not** for a technical task, a bug, an endpoint, a migration, a UI
change or a one-off request.

> "Is this something the team would want to see as a main goal when reviewing the next
> sprints?" If not → Story/Task. **When in doubt, do not create an Epic.**

**Name:** a business capability or outcome, short and general; never a user story.
Good: "[OPS] Order status and fulfillment reporting". Bad: "The user wants to unlock the invoice".

**Domain tag** at the start of the name, exactly one per Epic. Fill this in for your organization:

| Tag | Domain |
|---|---|
| `[<TAG>]` | e.g. Customer-facing chatbot |
| `[<TAG>]` | e.g. Main web product: accounts and dashboard |
| `[<TAG>]` | e.g. Data platform and telemetry |
| `[<TAG>]` | e.g. Device firmware / integrations |
| `[SRE]` | Operating the platform (infra, versions, cost) |
| `[<TAG>]` | e.g. Internal support and operations processes |

If the work spans domains, it is several Epics or the scope is too broad. **Do not invent
domains**: ask. (An Epic without a tag is acceptable only if the team decided so and the reason
is written down.)

## Procedure for creating work

1. **Understand the request** without inventing scope: what, why, and whether it is a requirement,
   technical work, a bug or a question.
2. **Find the Epic:** `project = PROJ AND issuetype = Epic ORDER BY created DESC`. If several fit,
   choose by scope; if none does, create the issue **without an Epic** (cheaper than an extra Epic).
3. **Choose the type** with the table above, and create **the smallest issue** that represents
   the request. Subtasks only if the task cannot be done in one go.
4. The description carries acceptance criteria and output artifacts → `/acceptance-criteria`.

## Sprints

`PROJ` works in sprints. **Do not assign sprint or priority** unless asked: the team sets them.
A ticket can start refining outside the sprint and continue inside it.
What is in the open sprint: `project = PROJ AND sprint in openSprints()`.

**Ticket fields**: besides the description, "Acceptance criteria", "Output artifacts", "UI
design" and "Refinement checklist". Keys, which types have them and how to write them (in ADF,
not Markdown): [references/fields.md](references/fields.md).

## Ticket states

Use **a single workflow for all types** (Epic, Story, Task, Bug, Investigation, Subtask). Check
the exact names with `getTransitionsForJiraIssue` on any ticket and use them verbatim in
transitions and JQL. The recommended workflow:

| State | Category | Means | Exits via |
|---|---|---|---|
| `To Do` | To Do | Just created, not refined | triage → `Refining` |
| `Refining` | To Do | Criteria and output artifacts are being written | **ready to start** (the owner moves it; check with `/definition-of-ready`) → `Refined` |
| `Refined` | To Do | Spec approved; ready to plan | plan ready → `In Progress` |
| `In Progress` | In Progress | Being built, one lane per subtask | **deployed** where it is validated → `Validating` |
| `Validating` (formerly `QA`) | In Progress | Installed where it is validated (staging, or production behind a flag) | **validated and delivered** → `Done` · rework → `In Progress` · new criterion → `Refining` |
| `Done` | Done | Reached the customer | — |
| `Discard` | Done | Not going to be done, and nothing is left installed | — (how: [references/discard.md](references/discard.md)) |

**Transitions are free** (all global): Jira does not prevent starting without being ready;
`/definition-of-ready` checks that, not the workflow.

**Do not transition a ticket on your own** unless asked, and never from `Refining` to `Refined`.

## Blocking links

"A blocks B" means B cannot start until A is Done. Check the direction before creating one:
backlogs often contain Done tickets "blocked by" tickets that have not started (e.g. `PROJ-66`
blocked by `PROJ-67`) — a link created in the wrong direction. Point them out; do not fix them
without asking.
