---
name: gate-policies
description: The web team's checkpoint (gate) policies and how they map to Jira labels — ff-required / ff-none, rollout-plan, qa-dev / qa-stakeholder, design-needed / design-none and the "-ask" labels — which output artifact each one is derived from, which policy it activates at deploy, validation and delivery, the dangerous combinations, and which policies let an agent pass a checkpoint (Dependabot auto-merge, docs, tests, dashboards, revert). Use when labeling a ticket in refinement, when checking whether the labels match the spec, or when someone asks "which label do I use?", "does this need a flag?" or "can this merge on its own?". Invoke as /gate-policies [JIRA-KEY].
---

# gate-policies — the web team's labels and policies

Source: your team's working-agreement / process proposal, "Policies" and "Labels" sections (link it
here, with its version). What a checkpoint is: `/ticket-lifecycle` (which also says what is
automated). **Record the adoption state here** (e.g. "As of YYYY-MM-DD: no issue in `PROJ` carries
these labels yet") so nobody assumes enforcement that doesn't exist.

## Labels don't decide: they summarize the artifacts

Labels are derived from the **output artifacts** (`/acceptance-criteria`) and say which policy
applies. If a label contradicts the artifacts, **the artifacts win** and the label is corrected. A
web ticket carries one label from each family (FF, QA, design); `rollout-plan` is added on top.

| Label | Set it if an output artifact… | Activates | In one line |
|---|---|---|---|
| `ff-required` | runs in the runtime, is user-visible and needs approval | deploy-flagged · delivery-flag | Lands in production switched off; the owner turns it on (`/feature-flags`) |
| `ff-none` | runs in the runtime and needs no outside approval | deploy-standard | Live for everyone on merge. For bugs and low risk |
| `rollout-plan` | changes or deletes data, schema or a cross-repo contract | deploy-stepwise | One PR per step, data export first; a flag does not undo it (`/rollout-plan`) |
| `qa-dev` | is checked with data, logs or the screen | validation-evidence | The developer (or a validator agent, with evidence for each criterion) |
| `qa-stakeholder` | is judged by someone outside the team | validation-stakeholder | Passed by the named stakeholder, **never an agent** |
| `design-needed` | is a new screen or a visible change (linked from the "UI design" field) | design-first | The design subtask goes first; the rest waits for it |
| `design-none` | changes nothing visible | — | No design lane |
| `ff-ask`, `qa-ask`, `design-ask` | can't be derived from the spec | — | **Not ready** until the owner clarifies it (`/definition-of-ready`) |

When in doubt between `ff-none` and `ff-required` for **new behavior**, the label is `ff-ask`: with
`ff-none` the user sees it before anyone has validated it.

### Does it need a flag?

This is the only place where that is decided; `/gitflow` and `/feature-flags` point here. In the web
apps the merge deploys to production (`/gitflow`), so anything a user must not see on the day it is
merged gets `ff-required`:

- the UI shows something half-done, or it is pending validation in staging;
- a new endpoint that a client must not use yet;
- behavior the business wants to switch on at a given date or after approving it;
- a change coordinated across repos that haven't all deployed yet.

A change that breaks a contract or deletes data is **not** solved with a flag: `rollout-plan`.
How a flag is created, turned on and retired in each repo: `/feature-flags`.

## Dangerous combinations

- `qa-stakeholder` + `ff-none`: the user sees it before the stakeholder says yes.
- `qa-dev` on something that needs an opinion: the same team writes, reviews and accepts.
- `rollout-plan` + `ff-required` as if the flag protected the migration: it doesn't.

## Policies per checkpoint

"Ready to start" has no policy: the owner always passes it (`/ticket-lifecycle`).

| Checkpoint | Decides by default | An agent may pass it if… | Passes empty if… |
|---|---|---|---|
| Deploy | A reviewer from the repo's team | An auto-merge policy allows it (below) | No artifact goes to a repo or the runtime |
| Validation | Developer (`qa-dev`) or stakeholder (`qa-stakeholder`) | `qa-dev` and every criterion has automatic evidence | The artifacts don't run (docs, tests): CI already checked them |
| Delivery | Ticket owner | No flag and validation accepted | — |

Proposed **auto-merge** policies at deploy — all require green CI and, except dashboards and revert,
approval from the Reviewer (an agent in a separate session):

| Policy | Met if the PR… |
|---|---|
| deploy-dependabot | is from `dependabot[bot]` and only changes dependencies |
| deploy-docs | only touches `docs/`, `README` or `AGENTS.md` |
| deploy-tests | only adds or changes tests |
| deploy-dashboards | only changes Grafana dashboard or alert JSON |
| deploy-revert | reverts the last merged PR and there is a linked open alert |

## Rules for a good policy

- Conditions a machine can check; scope limited by artifact type; reversible.
- Record on the ticket or PR which policy was applied; someone reviews a sample every week.
- **Not acceptable:** "the LLM says it's fine" as the only condition, delegating "ready to start",
  or migrations that delete data, secrets or permissions.
- Policies are artifacts: they change through a PR approved by the checkpoint's owner.

**Unverified:** whether GitHub branch protection accepts a bot's approval as a required review.
Don't promise auto-merge until you've checked it in the repo.
