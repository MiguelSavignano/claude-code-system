---
name: gate-policies
description: The web team's checkpoint (gate) policies and how they map to ticket labels (Jira labels, GitHub labels, Asana tags or custom fields) — ff-required / ff-none, rollout-plan, qa-dev / qa-stakeholder, design-needed / design-none and the "-ask" labels — which output artifact each one is derived from, the catalog of policies they activate at deploy, validation and delivery (and the auto-merge policies that let an agent pass a checkpoint: Dependabot, docs, tests, dashboards, revert), the dangerous combinations, and which other skill covers each related step. Use when labeling a ticket in refinement, when checking whether the labels match the spec, or when someone asks "which label do I use?", "does this need a flag?" or "can this merge on its own?". Invoke as /gate-policies [TICKET-KEY].
---

# gate-policies — the web team's labels and policies

Source: your team's working-agreement / process proposal, "Policies" and "Labels" sections (link it
here, with its version). **Record the adoption state here** (e.g. "As of YYYY-MM-DD: no issue in
`PROJ` carries these labels yet") so nobody assumes enforcement that doesn't exist.

This skill has three parts, kept apart on purpose: **labels** (what goes on the ticket), **policies**
(the rule that decides who passes a checkpoint), and **related skills** (where the *how* lives).

## 1. Labels: they summarize the artifacts

Labels are derived from the **output artifacts** and say which policy applies. If a label
contradicts the artifacts, **the artifacts win** and the label is corrected. A web ticket carries
one label from each family (FF, QA, design); `rollout-plan` is added on top.

| Label | Set it if an output artifact… | Activates policy |
|---|---|---|
| `ff-required` | runs in the runtime, is user-visible and needs approval | `deploy-flagged` · `delivery-flag` |
| `ff-none` | runs in the runtime and needs no outside approval | `deploy-standard` |
| `rollout-plan` | changes or deletes data, schema or a cross-repo contract | `deploy-stepwise` |
| `qa-dev` | is checked with data, logs or the screen | `validation-evidence` |
| `qa-stakeholder` | is judged by someone outside the team | `validation-stakeholder` |
| `design-needed` | is a new screen or a visible change (linked from the "UI design" field) | `design-first` |
| `design-none` | changes nothing visible | — |
| `ff-ask`, `qa-ask`, `design-ask` | can't be derived from the spec | — (ticket is **not ready**) |

When in doubt between `ff-none` and `ff-required` for **new behavior**, the label is `ff-ask`: with
`ff-none` the user sees it before anyone has validated it.

### Does it need a flag?

This is the only place where that is decided. In the web apps the merge deploys to production, so
anything a user must not see on the day it is merged gets `ff-required`:

- the UI shows something half-done, or it is pending validation in staging;
- a new endpoint that a client must not use yet;
- behavior the business wants to switch on at a given date or after approving it;
- a change coordinated across repos that haven't all deployed yet.

A change that breaks a contract or deletes data is **not** solved with a flag: `rollout-plan`.

### Dangerous combinations

- `qa-stakeholder` + `ff-none`: the user sees it before the stakeholder says yes.
- `qa-dev` on something that needs an opinion: the same team writes, reviews and accepts.
- `rollout-plan` + `ff-required` as if the flag protected the migration: it doesn't.

## 2. Policies

"Ready to start" has no policy: the owner always passes it.

### Policies activated by labels

| Policy | Checkpoint | Rule |
|---|---|---|
| `deploy-standard` | Deploy | A reviewer from the repo's team approves; on merge it is live for everyone |
| `deploy-flagged` | Deploy | Same review; it lands in production with the flag **off** |
| `deploy-stepwise` | Deploy | One PR per step, data export first; each step reviewed on its own |
| `design-first` | Before build | The design subtask is accepted before the rest of the ticket starts |
| `validation-evidence` | Validation | Developer, or a validator agent with evidence for **each** criterion |
| `validation-stakeholder` | Validation | The named stakeholder passes it, **never an agent** |
| `delivery-flag` | Delivery | The ticket owner turns the flag on |

### Auto-merge policies (proposed) — an agent passes Deploy

All require green CI and, except `deploy-dashboards` and `deploy-revert`, approval from the
Reviewer (an agent in a separate session).

| Policy | Met if the PR… |
|---|---|
| `deploy-dependabot` | is from `dependabot[bot]` and only changes dependencies |
| `deploy-docs` | only touches `docs/`, `README` or `AGENTS.md` |
| `deploy-tests` | only adds or changes tests |
| `deploy-dashboards` | only changes Grafana dashboard or alert JSON |
| `deploy-revert` | reverts the last merged PR and there is a linked open alert |

### Who passes each checkpoint

| Checkpoint | Decides by default | An agent may pass it if… | Passes empty if… |
|---|---|---|---|
| Deploy | A reviewer from the repo's team | An auto-merge policy applies | No artifact goes to a repo or the runtime |
| Validation | Developer (`qa-dev`) or stakeholder (`qa-stakeholder`) | `validation-evidence` and every criterion has automatic evidence | The artifacts don't run (docs, tests): CI already checked them |
| Delivery | Ticket owner | No flag and validation accepted | — |

### Rules for a good policy

- Conditions a machine can check; scope limited by artifact type; reversible.
- Record on the ticket or PR which policy was applied; someone reviews a sample every week.
- **Not acceptable:** "the LLM says it's fine" as the only condition, delegating "ready to start",
  or migrations that delete data, secrets or permissions.
- Policies are artifacts: they change through a PR approved by the checkpoint's owner.

**Unverified:** whether GitHub branch protection accepts a bot's approval as a required review.
Don't promise auto-merge until you've checked it in the repo.

## 3. Related skills

| For… | Skill |
|---|---|
| What a checkpoint is and which are automated | `/ticket-lifecycle` |
| The output artifacts labels are derived from | `/acceptance-criteria` |
| Blocking a ticket with an `-ask` label | `/definition-of-ready` |
| How a merge reaches production | `/gitflow` |
| Creating, turning on and retiring a flag (`deploy-flagged`, `delivery-flag`) | `/feature-flags` |
| Writing the steps of a `deploy-stepwise` change | `/rollout-plan` |
