---
name: ticket-lifecycle
description: How to work with AI across the ticket lifecycle — Jira and its states, GitHub and its PRs, QA with acceptance criteria and policies for changing state, with the trail on the ticket; the ticket is the state, every ticket declares its output artifacts, and it passes four gates (ready to start, deployment, validation and delivery), each with an owner and a policy saying who may pass it. Includes the map of which skill covers each step. Use when someone asks "how do we work with AI", "which gate is next", "who can pass this", or before designing a skill or agent that moves tickets. Invoke as /ticket-lifecycle.
---

# ticket-lifecycle — the ticket lifecycle on one page

Source: your team's process proposal (record where it lives in your team wiki). **Treat this as
a proposal until your team adopts it, and state what is automated and what is not** (see the end).

## The idea

Bring together what a Scrum team already does — **Jira and its states, GitHub and its PRs, QA
with acceptance criteria and state changes driven by policies** — and write down the agreements
that used to be made verbally. Once written, AI can step in to refine, code and validate, alone
or alongside a person, and everything leaves a trail in one place: the Jira ticket. It does not
forbid vibe coding: inside `In Progress` you work however you like; the lifecycle says which
state the ticket is in, which policy applies and what counts as delivered.

What matters is not the procedure but **what ends up done: the artifacts** (a PR, an applied
migration, the email that reaches the account manager, a written decision).

1. **The ticket is the state.** Its description and fields (criteria, artifacts, design) are the
   spec; its comments, the trail; its state, the only thing everyone looks at.
2. **Every ticket declares its output artifacts**: what has to exist at the end, where and in
   what state. That list determines what is checked at each gate → `/acceptance-criteria`.
3. **Every state change is an event** that whoever is responsible reacts to (person or agent).
   One skill = one event, one task; none carries the whole flow.

## The four gates

| Gate | The question | Jira | Can an agent pass it? |
|---|---|---|---|
| **1 · Ready to start** | Do we agree on when it is done? | Refining → Refined | **Never.** The ticket owner passes it → `/definition-of-ready` checks it |
| **2 · Deployment** | Is it safe to put it where it will be validated? | In Progress → Validating | Only with a policy (e.g. Dependabot auto-merge) |
| **3 · Validation** | Does it do what was agreed, under real conditions? | Validating | Only with a policy and evidence; never if a stakeholder judges it |
| **4 · Delivery** | Has it reached the customer? | Validating → Done | Only with a policy |

- **In Progress** is the agent's loop: one subtask, one PR, CI, merge, and CI/CD deploys to where
  it is validated (`/gitflow`, `/pull-request`, `/ci-cd`).
- **Validation** is partly done by an agent following a method: each criterion against its
  artifact, with the evidence on the ticket. **A person enters the loop** when a policy is not met
  or a decision is needed, and the decision is written on the ticket. If a new output artifact
  appears, it is a new criterion: back to `Refining` (`/acceptance-criteria`).
- **Delivery is the real one.** Being in production is not being delivered: it may be deployed
  behind a flag with no customer using it.

A gate **is not skipped: it is passed**, in one of three ways, and all leave a trail on the
ticket: **empty** (no output artifact crosses it: "Deployment: no artifacts to deploy"), **by
policy** (which policy was applied and its evidence) or **by decision** (who and when).

Each gate has an **owner** (a person or team, accountable for the decision) and a **policy** the
owner approves. Policies show up in Jira as **labels**, derived from the output artifacts; if they
clash, the artifacts win. **Each department defines its policies and labels in its own plugin**;
the web team's are in `/gate-policies`.

## Which skill covers each step

| Step | Skill |
|---|---|
| Create the ticket in `PROJ` with the right type; board states and transitions | `/jira-task` |
| Find the context before refining (wiki, code, PRs, data) | `/refinement-context` |
| Write acceptance criteria and output artifacts | `/acceptance-criteria` |
| Can it move to Refined? | `/definition-of-ready` |
| Which labels it carries and which policy they trigger (web) | `/gate-policies` |
| Shipping on web: branch, PR, flag, staged rollout | `/gitflow`, `/pull-request`, `/feature-flags`, `/rollout-plan` |

## Rules for any skill or agent that moves tickets

- **Only pass a gate with a policy**, and write which one was applied. Without a policy, wait for the owner.
- **Safe if the event arrives twice**: before acting, check whether its work is already done.
- **No loops**: ignore events triggered by another agent, except the expected ones.
- **One owner per ticket section** (whoever refines owns the spec; whoever plans, the plan; whoever builds, their PR).
- **Trail and budget**: one comment per run; two retries and then notify the owner.

## What exists and what does not (fill this in for your organization)

| Piece | Status |
|---|---|
| The workflow states exist in `PROJ`; does Jira block any jump? (`/jira-task`) | e.g. states exist, no jump is blocked |
| Policy labels in use (`/gate-policies`) | e.g. not yet |
| Webhooks or agents connected to state changes | e.g. none: skills are launched by hand |

Do not present the proposal as if it already worked. If someone asks "does an agent already do
this?", answer with what this table says today.
