---
name: definition-of-ready
description: Checks whether a ticket (Jira, GitHub Projects, Asana or any tracker) in Refining can move to Refined (the "Definition of Ready") — a pass/fail list with evidence taken only from the ticket: in Refining, owner and Epic, goal, checkable acceptance criteria, output artifacts, repos, expensive decisions closed, policy labels without "-ask", no open blocker (including reverse "blocks" links) and design if something visible changes; the rule for which open questions may wait; and a report table (item | pass/fail | evidence). It never moves the ticket: a person always passes this step. Use in refinement, in sprint planning, or when someone asks "can I start PROJ-123?" or "which tickets are ready?". Invoke as /definition-of-ready [TICKET-KEY | sprint | refining].
---

# definition-of-ready — can this ticket move to Refined?

Source: <your working agreement> · Checked YYYY-MM-DD. Whether your team has adopted this step
and what the tracker enforces: `/ticket-lifecycle` ("What exists").

**Type:** check. You read, compare with the list and give a verdict. **You never move the
ticket**, not even if asked: the ticket owner always passes "ready to start" (`/ticket-lifecycle`).
States, transitions and refining outside the sprint: `/task-tracker`.

## Procedure

1. With a key: read the ticket — title, description, state, type, labels, parent Epic, owner,
   blocking links in both directions, comments, and the criteria, artifacts and design fields
   (where each lives: `/task-tracker`). With `refining`: every ticket in `Refining`. With
   `sprint`: the current sprint's tickets in `Refining`; list `To Do` ones apart as "triage first".
2. **Judge only what is written in the ticket.** What you know from this conversation and is not
   in the ticket does not count: the next person who reads it will not have it.
3. For each blocking link, check the other ticket's state.
4. Fill in the list and give the verdict.

## The list

| # | Item | Passes if… |
|---|---|---|
| 1 | In Refining | State is `Refining`. A `To Do` ticket fails: it goes through triage first (`/task-tracker`) |
| 2 | Owner and Epic | It has an owner, and hangs from an Epic with a domain tag (`/task-tracker`) or says why it has none |
| 3 | Goal | It says what is to be achieved and for whom |
| 4 | Acceptance criteria | Present and checkable per `/acceptance-criteria`, including what its ticket type requires |
| 5 | Output artifacts | Written per `/acceptance-criteria` ("Where to test" row included); every criterion has an artifact and every artifact a criterion |
| 6 | Repos and prerequisites | It names the repos/systems it touches and the manual steps (cloud consoles, third-party portals) |
| 7 | Expensive decisions closed | Data identity, permissions, legal or contractual rules and cross-repo contracts are decided; open questions follow the rule below |
| 8 | Policy labels | Its team's labels are set and none ends in `-ask` (web: `/gate-policies`). No labels in the team: "N/A — the team has no policy labels" |
| 9 | No open blocker | No `is blocked by` to a ticket that is not Done, and no reverse "blocks" link pointing at this ticket from an unfinished one (`/task-tracker`) |
| 10 | Design | If something visible changes, the design field links the design and says what changes (`/app-design`); otherwise N/A |

**Open questions** (comments in the format of `/acceptance-criteria`): one may wait if its answer
is configuration (copy, fields, thresholds in settings) and names who answers it. If the answer
changes code, schema or permissions, it must be answered before Refined: item 7 fails.

Each item is **Pass**, **Fail** or **N/A** (with the reason). No "partial": if in doubt, it
fails, and say what is missing. Criteria still in the description of an older ticket count;
propose moving them to their field.

## Report

```markdown
### PROJ-123 · <title> — NOT READY (3 failures)
| # | Item | Result | Evidence / what is missing |
|---|---|---|---|
| 4 | Acceptance criteria | Fail | "Works well" does not say where it is checked |
…
**To pass:** 1. … 2. … (blockers first)
**Decides:** <ticket owner>
```

With several tickets, start with a summary table (key, verdict, failures), tickets without
criteria first, then the detail only for those that fail.

## What you do not do

- Move the ticket, in any case. Change labels or description only if asked, proposing the text
  with `/acceptance-criteria` and the labels with `/gate-policies` first. Comment the report on
  the ticket only if asked.
- Fix blocking links: point them out.
- Mark a ticket ready because "it is understood": if it is not written, it fails.
