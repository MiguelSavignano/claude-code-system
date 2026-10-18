---
name: acceptance-criteria
description: How to write the acceptance criteria (AC) and output artifacts of a Jira ticket — what makes a criterion testable, the typical failures, what each ticket type needs (story, bug, enabler, investigation), the rule that every criterion is checked against at least one artifact, and the template for the ticket fields. With one good and one bad example ticket. Use when writing or reviewing a ticket's AC, in refinement, or when someone asks "is this a good criterion?". Invoke as /acceptance-criteria [JIRA-KEY].
---

# Acceptance criteria and output artifacts

A ticket without testable criteria is where AI fills the gap with something plausible. Measure
it in your own backlog: count how many recent tickets carry criteria that someone else could
check without asking the author. A low ratio is the signal to fix refinement first.

| | Says | Example |
|---|---|---|
| **Acceptance criterion** | What has to be **true** | "The supplier list is sorted without taking verification status into account" |
| **Output artifact** | What has to **exist**, where and in what state | "The production database without the `suppliers.verified` column" |

**Rule:** every criterion is checked against at least one artifact, and every artifact has at
least one criterion. The artifact that matters is almost never the PR (the means) but the result:
the email that arrives, the clean database, the written decision.

## A criterion is testable if…

1. **It says where to look**: which environment, which account, which screen, which table, which log.
2. **It has an observable result**: something you can see, count or query. Not "works well".
3. **Someone who did not build it can check it**, without asking the author.
4. **It is one per line**, as a `- [ ]` checklist. If it contains "and", it is probably two.
5. **It says what must NOT happen** when the risk is there (regressions, another customer's data).

Typical failures, and how to fix them:

| Bad | Why | Better |
|---|---|---|
| "Improve list performance" | No threshold | "`GET /accounts/:id/orders` responds in < 500 ms with 1,000 orders in staging" |
| "The user can see the report" | Which user, what data? | "A company user sees only their company's orders; a user from another company gets 403" |
| "It is tested" | It is a means, not a result | Remove it: tests are Definition of Done, not an AC |
| "Same as in the other app" | Depends on someone's memory | Link the screen or copy the expected behavior |

## What each ticket type needs

| Type | The criteria state… |
|---|---|
| **Story** | The visible behavior, agreed with the stakeholder, in its real context |
| **Bug** | Steps to reproduce, expected result and actual result |
| **Enabler** (Task/Bug under platform, data or SRE Epics) | A measurable technical state: "RDS on PostgreSQL 15 in staging and production" |
| **Investigation** | The question, who decides, and that the decision is written down and the new tickets linked |

## Examples (invented, each showing one teaching point)

- **PROJ-115 — good.** "On `acme-corp` (a real customer account, **not** a `test_00x` fixture):
  a payment failure is triggered and the support assistant reports it; once it is resolved, the
  user writes "check again" and the assistant says it is no longer active" and "In Loki, the turn
  shows a new call to the billing API (`GET /payments/failures`), not a cache hit". It says
  where, what you see and how it is tested. **What it lacks** is the output artifacts section
  (the changed tool definition, the corrected docs).
- **PROJ-12 — cannot be started.** Title "…basic report of my orders. number of orders",
  description "As a customer I want to see my order report." No where, no which data, no who
  "customer" is, no artifacts.

## Where each thing goes

Each in its own ticket field, not in the description (keys and format: `/jira-task`):

| Field | Content |
|---|---|
| Acceptance criteria | One checkbox per criterion: `<where> · <what happens> · <what you see / query>` |
| Output artifacts | Table: artifact · where it lives · final state · criterion that checks it (e.g. "PR in <repo> · GitHub → production · merged and deployed · AC 1") |
| Comment | Each open question: `<question> — answered by <person>. Does it change code, schema or permissions? yes/no` |

They are written in ADF (native checkboxes and table); in Markdown the checkboxes end up as text.

**The first row of the artifacts is always "Where to test"**: the URL of the environment where it
is validated (or "local, test bench" if it is not deployed), the account or test data, and the
role or flag required. Whoever validates starts there; without that row nobody can check anything.
Staging URLs and how to log in: `/test-apps`, if your team has it. Never credentials: say how to
get them.

| Artifact | Where it lives | Final state | Criterion |
|---|---|---|---|
| **Where to test** | `https://<staging>/accounts/acme-corp/invoices/` · flag `INVOICE_FORM_ENABLED` · authorized test email | Accessible to whoever validates | All |

## A new criterion after validation

This is normal: many things only show up when you try them. The signal is **a new output
artifact**, something that was not going to exist before. Example: in PROJ-59, after a meeting
with the Finance team, "downloadable invoice summary" was not "one more field": it was a new
artifact (the PDF summary the customer receives) and therefore a new criterion. It stayed in a
comment and the ticket was closed as Done with the criteria untouched — the anti-pattern to avoid.

- **Same ticket, back to `Refining`**, if delivering it makes no sense without it. Add it to the
  criteria field without deleting anything, with when and who decided, and check again that it is
  ready (`/definition-of-ready`):
  `- [ ] Customer portal · the customer downloads the invoice summary · the PDF includes it (added after validation, <date>, decided by Finance)`
- **New linked ticket**, and the current one continues to delivery, if it is an improvement that
  can come later.
- The ticket owner decides, and the decision is written in the ticket. How to mark what is
  removed: `/jira-task` (discard).

## How to use it

- If you are given a key, read the ticket with `getJiraIssue` and **propose** the text; do not
  edit the ticket unless asked.
- **Context comes first**: what already exists in the wiki, the code and the data → `/refinement-context`.
- **Do not invent** criteria or thresholds: if a fact is missing, it is an open question (a
  comment) with who answers it.
- An open question can wait if its answer is configuration (fields, copy, thresholds in
  settings). If it changes code, schema or permissions, it is answered before starting.
- Labels derived from the artifacts → `/gate-policies`. If the ticket is ready to start →
  `/definition-of-ready`.
