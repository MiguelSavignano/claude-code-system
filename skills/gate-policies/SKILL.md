---
name: gate-policies
description: The web team's ticket labels and the gate policies they activate — ff-required / ff-none, rollout-plan, qa-dev / qa-stakeholder, design-needed / design-none and the "-ask" labels — which output artifact each label is derived from, when a change needs a flag, the dangerous label combinations, each policy by gate (deployment, validation, delivery) with who passes it and when an agent may, and the proposed auto-merge policies (Dependabot, docs, tests, dashboards, revert) with their limits. Use when labeling a ticket in refinement, when checking whether the labels match the spec, or when someone asks "which label do I use?", "does this need a flag?" or "can this merge on its own?". Invoke as /gate-policies [TICKET-KEY].
---

# gate-policies — the web team's labels and policies

Source: your team's working agreement, "Labels" and "Policies" sections — link it here with its
version and **Checked YYYY-MM-DD**, plus the adoption state (e.g. "no ticket carries these labels
yet"), so nobody assumes enforcement that doesn't exist. What a gate is, how one is passed and the
rules every policy must meet: `/ticket-lifecycle`.

## 1. Labels

A web ticket carries one label per family (FF, QA, design); `rollout-plan` is added on top. Derive
each from the output artifacts (`/acceptance-criteria`).

| Label | Set it when an output artifact… | Activates |
|---|---|---|
| `ff-required` | runs in production and users must not see it the day it merges | `deploy-flagged` · `delivery-flag` |
| `ff-none` | runs in production and can be live for everyone on merge | `deploy-standard` · `delivery-standard` |
| `rollout-plan` | changes or deletes data, schema or a cross-repo contract | `deploy-stepwise` |
| `qa-dev` | is checked with data, logs or the screen | `validation-evidence` |
| `qa-stakeholder` | is judged by someone outside the team | `validation-stakeholder` |
| `design-needed` | is a new screen or a visible change | build order: the design subtask is accepted first |
| `design-none` | changes nothing visible | — |
| `ff-ask` · `qa-ask` · `design-ask` | can't be derived from the spec yet | ticket is **not ready** (`/definition-of-ready`) |

### Does it need a flag?

Decided only here. The merge deploys to production, so `ff-required` when:

- the UI is half-done, or still pending validation in staging;
- a new endpoint that clients must not use yet;
- the business turns it on at a date or after approving it;
- a change spread across repos that haven't all deployed yet.

New behavior and unsure → `ff-ask`, not `ff-none`. A flag does not protect a change that breaks a
contract or deletes data: that is `rollout-plan`.

### Dangerous combinations

- `qa-stakeholder` + `ff-none`: users see it before the stakeholder says yes.
- `qa-dev` on something that needs an opinion: the same team builds, reviews and accepts.
- `rollout-plan` + `ff-required` as if the flag protected the migration: it doesn't.

## 2. Policies by gate

"Ready to start" has no policy: the ticket owner always passes it.

| Gate | Policy | Who passes it | An agent may pass it if… |
|---|---|---|---|
| Deployment | `deploy-standard` | A reviewer from the repo's team; on merge it's live for everyone | an auto-merge policy (§3) applies |
| Deployment | `deploy-flagged` | Same review; lands in production with the flag **off** | an auto-merge policy applies |
| Deployment | `deploy-stepwise` | One PR per step, each reviewed on its own (`/rollout-plan`) | **never** |
| Validation | `validation-evidence` | The developer, or a validator agent | every criterion has automatic evidence on the ticket |
| Validation | `validation-stakeholder` | The named stakeholder | **never** |
| Delivery | `delivery-flag` | The ticket owner turns the flag on in production (`/feature-flags`) | **never** |
| Delivery | `delivery-standard` | The ticket owner | validation is accepted |

A gate passes **empty** when no artifact crosses it: Deployment if nothing goes to a repo or the
runtime; Validation if the artifacts don't run (docs, tests) and CI already checked them.

## 3. Auto-merge policies (proposed)

An agent may merge, passing Deployment, only when **all** hold: green CI, approval from a reviewer
agent in a separate session, the PR names the policy, and **no** migration, secret, permission,
workflow or agent-instruction file is touched (`AGENTS.md`, `CLAUDE.md`, `.claude/`).

| Policy | Met if the PR… |
|---|---|
| `deploy-dependabot` | is from `dependabot[bot]` and only changes dependency manifests and lockfiles |
| `deploy-docs` | only touches `docs/` or `README*` |
| `deploy-tests` | only adds or changes tests |
| `deploy-dashboards` | only changes Grafana **dashboard** JSON (not alert rules: those can silence alerts) |
| `deploy-revert` | is a `git revert` of the last merged PR, that PR had no migration, and a linked alert is open |

**Unverified:** whether branch protection accepts a bot's approval as a required review. Check it in
the repo before promising any auto-merge.

## 4. Related skills

| For… | Skill |
|---|---|
| Gates, how one is passed, rules every policy must meet | `/ticket-lifecycle` |
| Output artifacts the labels come from | `/acceptance-criteria` |
| An `-ask` label blocks Refined | `/definition-of-ready` |
| Creating, enabling and retiring a flag | `/feature-flags` |
| Ordering a `deploy-stepwise` change | `/rollout-plan` |
