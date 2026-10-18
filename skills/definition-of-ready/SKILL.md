---
name: definition-of-ready
description: Checks whether a Jira ticket can move from Refining to Refined (the "Definition of Ready"), whether or not it is in the sprint — a pass/fail list with evidence taken only from the ticket: testable criteria, output artifacts, repos, expensive decisions closed, policy labels without "-ask", owner, Epic and blockers (including reversed "blocks" links). It does not move the ticket: a person always takes this step. Use in refinement, in sprint planning, or when someone asks "can I start PROJ-123?" or "which tickets are ready?". Invoke as /definition-of-ready [JIRA-KEY | sprint | refining].
---

# definition-of-ready — can this ticket move to Refined?

**Type:** check. You read, compare with the list and give a verdict. **You never transition the
ticket**: this step is not delegated (see `/ticket-lifecycle`). Comment the result on the ticket
only if asked.

## Context

- It is the `Refining` → `Refined` step (states and transitions: `/jira-task`). Jira does not
  enforce it: only this check does. A ticket that jumps to `In Progress` without going through
  `Refined` has skipped this check.
- Refinement can start outside the sprint and continue inside it: being in the sprint is not a
  requirement for being ready.
- While a team has not yet adopted policy labels, almost every ticket fails item 8 (status:
  `/gate-policies`). Say so plainly; do not hide it.

## Procedure

1. With a key: `getJiraIssue` (cloudId in `/jira-task`) with `summary, description, status,
   issuetype, labels, parent, assignee, issuelinks, comment`, the sprint and the criteria,
   artifacts and design fields (keys in `/jira-task`).
   With "refining": `project = PROJ AND status = Refining`; with "the sprint":
   `project = PROJ AND sprint in openSprints() AND status in ("To Do", Refining)`. Repeat per ticket.
2. **Judge only what is written in the ticket.** What you know from this conversation and is not
   in the ticket does not count: the next person who reads it will not have it.
3. For each `is blocked by`, check the other ticket's status.
4. Fill in the list and give the verdict.

## The list

| # | Check | Passes if… |
|---|---|---|
| 1 | In refinement | It is in `Refining` (or in `To Do`, if it is being refined now). The sprint does not count |
| 2 | Owner and Epic | It has an `assignee`, and hangs from an Epic with a domain tag (`/jira-task`) or the description says why it has none |
| 3 | Goal | It says what is to be achieved and for whom |
| 4 | Acceptance criteria | The "Acceptance criteria" field has them and they are testable per `/acceptance-criteria` (where, what you see, one per checkbox) |
| 5 | Output artifacts | The "Output artifacts" field starts with "Where to test" (URL or test bench, test data), every criterion has an artifact and every artifact a criterion |
| 6 | Repos and prerequisites | It names the repos/systems it touches and the manual steps (cloud console, third-party portals) |
| 7 | Expensive decisions closed | Data identity, permissions, legal or warranty rules and cross-repo contracts are decided. Remaining open questions are configuration and have someone to answer them |
| 8 | Policy labels | Its team's labels are set and none ends in `-ask` (web: `/gate-policies`) |
| 9 | No open blocker | No `is blocked by` pointing to a ticket that is not Done, and no reversed link (see `/jira-task`) |
| 10 | Design | If something visible changes, "UI design" links the canvas and says what changes (`/app-design`); otherwise not applicable |

Item 4 includes what its ticket type requires per `/acceptance-criteria`. In tickets created
before the dedicated fields existed, the criteria may be in the description: it counts, and
propose moving them to their field. For the sprint, the ones without criteria come out with
`"Acceptance criteria" is EMPTY`.

Each item is **Pass**, **Fail** or **Not applicable** (with the reason). No "partial": if in
doubt, it fails, and say what is missing.

## Response format

```markdown
### PROJ-123 · <title> — NOT READY (3 failures)
| # | Check | Result | Evidence / what is missing |
|---|---|---|---|
| 4 | Acceptance criteria | Fail | "Works well" does not say where it is checked |
…
**To pass:** 1. … 2. … (in order: blockers first)
**Decides:** <ticket owner>
```

With several tickets, start with a summary table (key, verdict, number of failures) and then the
detail only for those that fail.

## What you do not do

- You do not transition, or change labels or the description, unless asked. If asked, propose
  the text with `/acceptance-criteria` and the labels with `/gate-policies` first.
- You do not fix blocking links on your own: you point them out.
- You do not mark a ticket ready because "it is understood": if it is not written, it fails.
