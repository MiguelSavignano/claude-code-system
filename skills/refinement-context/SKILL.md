---
name: refinement-context
description: Where to look for a ticket's context while refining it, and how to write it down — which source answers each question (the team wiki for decisions and proposals, the code and GitHub PRs for how things work today, the task tracker (Jira, GitHub Projects, Asana…) for previous work, read-only database and log tools for production data, the document store for drawings and release files), the rules (every fact with source and date, verified or inference, contradictions written down, only SELECT and counts in production, a search limit) and the "Context" section left in the description. With two example tickets. Use when starting to refine a ticket, before writing its criteria, or when someone asks "what already exists for this?" or "how does X work today?". Invoke as /refinement-context [TICKET-KEY].
argument-hint: "[TICKET-KEY]"
---

# refinement-context — what already exists before deciding what to do

A ticket is refined on what exists, not on what someone remembers. The criteria
(`/acceptance-criteria`) come out of this context; without it, AI fills the gap with something plausible.

## Which source answers each question

| Question | Where | With what |
|---|---|---|
| What was already decided? Is there a proposal or architecture doc? | Your team wiki, especially its architecture/proposals section | A wiki search tool / MCP, if available (search, then read the page) |
| How does it work today? | The repo's code and its `AGENTS.md` / `CLAUDE.md` | The local clone or `gh` |
| Who touched this and why? | Recent PRs and issues in the area | `gh pr list --search`, `gh issue list` |
| Is there previous or related work? | Linked tickets, comments, the Epic, archived projects (read only) | The tracker's tool (`/task-tracker`) |
| How many are affected? What does the data say? | Production databases | A read-only SQL tool / MCP, if available |
| What happens at runtime? | Logs in Loki | A Loki query tool / MCP, if available (`/observability`) |
| Drawings, specs, release or configuration files | Your document store (e.g. SharePoint) | Its own tool or connector |

Data and log tools usually require elevated access. If they are not connected, say so and
continue with the rest.

## Rules

1. **Every fact with source and date**: the file and line, the wiki page, the PR, the query.
   And mark whether it is **verified**, an **inference** or **unverified**.
2. **Contradictions are written down, not resolved by eye.** The typical case: the wiki says one
   thing and the code another.
3. **In production, only `SELECT`.** SQL tools may be able to commit: no `INSERT`, `UPDATE`,
   `DELETE` or DDL.
4. **Counts, not rows.** No emails, names or secrets in the ticket: "60 orders of type
   `wholesale` require approval, 0 approved", not the list of orders.
5. **Search limit.** If it does not show up after 3 or 4 sources, it is an open question with who
   answers it, not an endless investigation.
6. **What you know from the conversation and is not in a source does not count**: the next person
   who reads the ticket will not have it.

## What stays in the ticket

A section in the description, before the criteria. Propose the text and do not edit the
description unless asked. Whatever remains unresolved goes as a comment on the ticket.

```markdown
## Context (<repos and sources>, <date>)
- Today: <how it works, with file or PR> — verified
- Data: <count and query> — verified
- Decided before: <wiki page or ticket>
- Contradiction: <source A says X; source B says Y>
- Unverified: <what is missing and who knows>
```

When done, propose ticking `Context` in the ticket's refinement checklist (`/task-tracker`);
do not tick it unless asked.

## Examples (invented, each showing one teaching point)

- **PROJ-115 — good.** "Current situation (analyzed in the `api` code, <date>)": it cites
  `chat_runner.py`, the actual cache TTL (5 s), and separates what is inference ("not confirmed
  in logs") from what is unverified (how often the upstream data refreshes). The criteria come
  straight out of it.
- **PROJ-59 — the contradiction that had to be written down.** The wiki proposal said "nothing
  implemented" and the `web` repo's `CLAUDE.md` already described the public invoice form. The
  ticket noted it under "Notes": exactly what has to be reconciled before estimating.

## What you do not do

- You do not decide the scope: the context informs; the owner agrees the criteria.
- You do not run anything that changes data or systems, in production or in staging.
- You do not copy whole documents into the ticket: summarize and link.
