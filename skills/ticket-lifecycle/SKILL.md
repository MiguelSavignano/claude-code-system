---
name: ticket-lifecycle
description: The ticket lifecycle on one page, for working with AI — the ticket is the state, every ticket declares its output artifacts, and it passes four gates (Ready to start, Deployment, Validation, Delivery), each with its question, states, an owner and a policy; the three ways to pass a gate (empty, by policy, by decision) with a trail on the ticket; the rules every policy must meet; the rules for agents that move tickets (idempotent, no loops, one owner per section, trail and retry budget); the map of which skill covers each step; and the adoption table to fill in. Use when someone asks "how do we work with AI", "which gate is next", "how is this gate passed", or before designing a skill or agent that moves tickets. Invoke as /ticket-lifecycle.
---

# ticket-lifecycle — the ticket lifecycle on one page

Source: <your working agreement> · Checked YYYY-MM-DD. **A proposal until your team adopts it.**

## The idea

Write down the verbal agreements so AI can refine, build and validate, alone or with a person.
What matters is **what ends up done: the artifacts** (a PR, an applied migration, a decision).

1. **The ticket is the state**: description and fields are the spec, comments the trail.
2. **Every ticket declares its output artifacts** (what, where, in what state); they decide what
   each gate checks → `/acceptance-criteria`.
3. **Every state change is an event** its owner (person or agent) reacts to; one skill = one step.

## The four gates

| Gate | Question | States | Owner by default |
|---|---|---|---|
| **1 · Ready to start** | Do we agree on when it is done? | `Refining` → `Refined` | The ticket owner (checked with `/definition-of-ready`) |
| **2 · Deployment** | Is it safe to put it where it will be validated? | `In Progress` → `Validating` | The repo's team (the merge is the gate) |
| **3 · Validation** | Does it do what was agreed, under real conditions? | `Validating` | The developer, or the stakeholder who judges it |
| **4 · Delivery** | Has it reached the customer? | `Validating` → `Done` | The ticket owner |

Whether an agent may pass a gate: the department's policies (web: `/gate-policies`). States,
transitions and whether the tracker blocks jumps: `/task-tracker`.

- **In Progress** is the build loop: one subtask, one branch, one PR, CI. **The merge is the
  Deployment gate**, passed per policy; CI/CD then deploys to where it is validated.
- **Validation**: each criterion against its artifact, evidence on the ticket. A person enters
  when no policy is met or a decision is needed. A new output artifact: back to `Refining`.
- **Delivery is the real one.** In production is not delivered: it may sit behind a flag no
  customer sees (`/feature-flags`).

## How a gate is passed, and what a policy must meet

A gate **is not skipped: it is passed**, in one of three ways, each with a trail on the ticket:
**empty** (no output artifact crosses it: "Deployment: no artifacts to deploy"), **by policy**
(which one, and its evidence) or **by decision** (who and when; always a person).

Each gate has an **owner** (accountable for it) who approves its **policies**. Each department
writes its policies, and the labels that select them, in its own plugin; labels are derived from
the output artifacts, and if they clash, the artifacts win. Every policy must have conditions **a
machine can check**, a **scope limited by artifact type** and a result that **can be reverted**.
**Not acceptable:** "the LLM says it's fine" as the only condition, delegating Ready to start, or
anything that deletes data or touches secrets or permissions. Policies change **through a PR
approved by the gate owner**; someone reviews a sample of applied policies regularly.

## Rules for agents that move tickets

- **An agent passes a gate only empty or by policy**, and writes which. By decision is always a
  person; with no policy that applies, wait for the owner.
- **Idempotent** (check whether the work is done before acting) and **no loops** (ignore events
  triggered by another agent, except the expected ones).
- **One owner per ticket section**: refiner → spec; planner → plan; builder → their PR.
- **Trail and budget**: one comment per run; after N retries (proposed default: 2), notify the owner.

## Which skill covers each step

| Step | Skill |
|---|---|
| Ticket types, states, transitions, fields; how your tracker does each | `/task-tracker` |
| Find the context before refining (wiki, code, PRs, data) | `/refinement-context` |
| Write acceptance criteria and output artifacts | `/acceptance-criteria` |
| Design a visible change before building it | `/app-design` |
| Gate 1: can it move to `Refined`? | `/definition-of-ready` |
| Labels and policies per gate (web) | `/gate-policies` |
| Build: branch, commits, PR, CI and deploy | `/gitflow`, `/pull-request`, `/ci-cd` |
| Gate 2 for risky changes; Gate 4 behind a flag | `/feature-flags`, `/rollout-plan` |
| Gate 3: smoke tests, logs and traces as evidence | `/test-apps`, `/observability` |
| Before designing a skill or agent that moves tickets | `/skill-design` (and the rules above) |

## Adoption state (fill this in for your organization)

| Piece | Status |
|---|---|
| Workflow states exist in `PROJ`; does the tracker block jumps? (`/task-tracker`) | e.g. states exist, no jump blocked |
| Policy labels in use (web: `/gate-policies`) | e.g. not yet |
| Webhooks or agents reacting to state changes | e.g. none: skills are launched by hand |

Never present the proposal as working: "does an agent already do this?" → answer from this table.
