---
name: acceptance-criteria
description: How to write the acceptance criteria (AC) and output artifacts of a ticket (in any tracker) — what makes a criterion checkable, the typical failures, what each ticket type needs (Story, Task including enablers, Bug, Investigation), the rule that every criterion is checked against at least one output artifact, the artifacts table whose first row is "Where to test", and the section template for the ticket description. With one good and one bad hypothetical example. It proposes the text; it does not edit the ticket unless asked. Use when writing or reviewing a ticket's AC, in refinement, or when someone asks "is this a good criterion?". Invoke as /acceptance-criteria [TICKET-KEY].
---

# Acceptance criteria and output artifacts

| | Says | Example |
|---|---|---|
| **Acceptance criterion** | What has to be **true** | "The supplier list is sorted without taking verification status into account" |
| **Output artifact** | What has to **exist**, where and in what state | "The production database without the `suppliers.verified` column" |

**Rule:** every criterion is checked against at least one artifact, and every artifact has at
least one criterion. The artifact that matters is the result, not the means: the email that
arrives, the clean database, the written decision. A PR is rarely it.

## A criterion is checkable if…

1. **It says where to look**: which environment, account, screen, table or log.
2. **It has an observable result**: something you can see, count or query. Not "works well".
3. **Someone who did not build it can check it**, without asking the author.
4. **It is one check per checkbox** (`- [ ]`). If it contains "and", it is probably two.
5. **It says what must NOT happen** when that risk exists (regressions, another customer's data).

| Bad | Why | Better |
|---|---|---|
| "Improve list performance" | No threshold | "`GET /accounts/:id/orders` responds in < 500 ms with 1,000 orders in staging" |
| "The user can see the report" | Which user, what data? | Two checkboxes: "A company user sees only their company's orders" · "A user from another company gets 403" |
| "It is tested" | A means, not a result | Remove it. If your team requires tests, that is a team criterion for every ticket, not an AC |
| "Same as in the other app" | Depends on someone's memory | Link the screen or copy the expected behavior |

## What each ticket type needs (types: `/task-tracker`)

| Type | The criteria state… |
|---|---|
| **Story** | The visible behavior, agreed with the stakeholder, in its real context |
| **Task** (enablers included) | A measurable technical state: "The database runs PostgreSQL 15 in staging" · "… in production" |
| **Bug** | Steps to reproduce, expected result and actual result; the fix is checked with the same steps |
| **Investigation** | The question, who decides, that the decision is written down, and that the follow-up tickets are linked |

## Examples (hypothetical, each showing one teaching point)

- **Good.** A support-assistant ticket whose criteria read:
  - [ ] Staging · on a real customer account (not a test account), trigger a payment failure · the assistant reports it
  - [ ] Staging · after the failure is resolved, the user writes "check again" · the assistant says it is no longer active
  - [ ] Staging · in the logs, that turn shows a new call to the billing API · not a cache hit

  Each says where, what happens and what you see. **It lacks** the output artifacts.
- **Bad, cannot be started.** Title "…basic report of my orders. number of orders", description
  "As a customer I want to see my order report." No where, no which data, no who "customer" is,
  no artifacts.

## The output artifacts table

Columns: artifact · where it lives · final state · criterion that checks it. **The first row is
always "Where to test"**: the URL of the environment where it is validated (or "local, test
bench"), the account or test data, and the role or flag required. Without it nobody can check
anything. Hosts come from the app repo or the team wiki; how to test is `/test-apps`. Never
credentials: say how to get them.

## Section template

Where each section is stored (dedicated field or description heading) and how to keep checkboxes
as checkboxes in your tool: `/task-tracker`. The content:

```markdown
## Acceptance criteria
- [ ] <where> · <what happens> · <what you see / query>

## Output artifacts
| Artifact | Where it lives | Final state | Criterion |
|---|---|---|---|
| **Where to test** | `https://<host>/<path>` · flag `<FLAG>` · <test account> | Accessible to whoever validates | All |
| Invoice email | Customer inbox (test account) | Received with the PDF attached | AC 1 |
```

Open questions go as comments: `<question> — answered by <person>`. Whether one can wait or must
be answered before starting: `/definition-of-ready`.

## A new criterion after validation

Normal; the signal is **a new output artifact**. Never leave it in a comment and close the ticket:
what happens to it (back to `Refining`, linked ticket, or discard) is in `/task-tracker`.

## How to use it

- With a key, read the ticket (`/task-tracker`) and **propose** the text; never edit it unless asked.
- **Context first**: what already exists in the wiki, the code and the data → `/refinement-context`.
- **Do not invent** criteria or thresholds: a missing fact is an open question with who answers it.
- Labels from the artifacts → `/gate-policies`. Ready to start → `/definition-of-ready`.
