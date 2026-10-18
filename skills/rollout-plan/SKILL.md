---
name: rollout-plan
description: How to order a change that alters data, schema or a cross-repo contract so each step is safe to deploy alone — expand → migrate → contract, one PR per step that is deployable and reversible on its own, compatibility shims and the rule for retiring them, why a feature flag does not protect a migration, the data-safety rules (export before a destructive step, dry-run, batches, verify counts; read-only queries, production only with explicit confirmation), and a plan template (step | PR | what changes | how to revert | verify). Use when a ticket carries the `rollout-plan` label, or before removing or renaming an endpoint, column, table, queue or config key that something else may use, or before a migration that alters or drops data. Invoke as /rollout-plan [TICKET-KEY].
argument-hint: "[TICKET-KEY]"
---

# rollout-plan — order the steps so each one deploys alone

**Type:** knowledge + plan template. A change that alters data, schema or a contract other repos
use is not one PR, it is a sequence of deploys, and **the order is the design**. This is how the
`rollout-plan` label's `deploy-stepwise` policy is met (label and policy: `/gate-policies`).

## Expand → migrate → contract

| Step | What it does | Why it is safe alone |
|---|---|---|
| **Expand** | Add the new endpoint/column/queue **next to** the old one | Nothing that works today stops working |
| **Migrate** | Move readers, writers and consumers to the new one — one consumer per PR; backfill data | The old path still exists; rolling back one consumer touches nothing else |
| **Contract** | Remove the old one, only when nothing uses it in any environment that matters | It is the only change in its deploy, so its rollback is just "put the old one back" |

- **One PR per step, each deployable and reversible on its own.** A PR that adds the new and
  removes the old has no rollback: if a consumer was missed, the only fix is under pressure.
- **Renaming or dropping a column is three migrations:** add + backfill → move readers/writers
  (dual-write if they deploy separately) → drop in a *later* deploy. Never drop in the release that
  stops writing to it.
- **Order across repos:** provider expands first; every consumer migrates (and is deployed and
  verified); then the provider contracts. How and when each repo actually deploys (on merge or
  manual): `/ci-cd`.
- **A feature flag does not protect a migration:** the schema change and the deleted data are
  there whether the flag is on or off (`/gate-policies`, dangerous combinations).

## Who uses the contract — find out before planning

"Nothing in this repo calls it" is not "nothing calls it": a contract has callers wherever its
consumers live. Say which of the two claims you are making.

- Grep **every repo you have access to** for the literal path, column, queue or key, and for the
  client that would call it (`*Client`; Rails example: `app/services/*_service.rb`).
- Decisions and integrations a repo's code never mentions (wiki, proposals): `/refinement-context`.
- A live caller you were about to break is a finding for the ticket owner, not something to route
  around silently.

## Compatibility shims — and retiring them

A shim (deprecated alias route, dual-write, old column kept in sync) is a legitimate bridge when
old and new must coexist. It is temporary only if it ships with:

1. **A named reason** — which consumer, which repo/PR is still on the old contract. "Backward
   compatibility" is not a reason.
2. **A checkable removal trigger** — a linked follow-up ticket in the consumer's repo, and a log
   line or metric on hits to the old path (`/observability`), so zero traffic is proven, not
   assumed. "Nobody complained" is not zero traffic.
3. **A comment at the shim** naming the consumer and the follow-up ticket.

**Retire the shims:** the contract step is part of the plan, not optional cleanup. The ticket is
not done while a shim it created has no follow-up ticket; if you can't measure traffic on the old
path, add that measurement before planning its removal — don't remove it blind or keep it forever.

## Data safety rules

- **Export or back up before any destructive step** (drop, delete, overwrite), with your team's
  backup/export mechanism — not an improvised dump. Staging data has value too.
- **Dry-run:** before an update or delete, run the same `WHERE` as a `SELECT COUNT(*)` and compare
  with what you expected. Write the expected number in the plan.
- **Batches** for large backfills, not one long transaction holding locks on a table others read.
- **Verify counts** after each data step (rows migrated = rows expected; no rows left on the old
  path) — that is the step's "verify" column.
- **Reversible migrations** by default (Rails example: `change` when Rails can invert it,
  otherwise explicit `up`/`down`). If one can't be reversed, say so in the PR description.
- **This skill only reads:** grepping repos and read-only counts are allowed; it runs no migration,
  update or delete. **Any query against production — even a read — needs the user's explicit
  confirmation**, and stays `SELECT`/counts only (`/refinement-context`). Pod access: `/kubectl`.

## Plan template

Write it in the ticket or the first PR's description before the first step merges:

```markdown
## Rollout plan — <TICKET-KEY>

Consumers found: <repo — where it calls (file/PR)>, … (searched: <repos>, <date>)
Destructive step(s): <step n> — export with <mechanism> before it

| Step | PR | What changes | How to revert | Verify (in the environment, not just CI) |
|---|---|---|---|---|
| 1 Expand | <repo>#<n> | Add `<new>` next to `<old>` | Revert the PR | `<new>` exists; old path unchanged |
| 2 Migrate | <consumer-repo>#<n> | Consumer uses `<new>` | Revert the PR; `<old>` still there | Deployed (`/ci-cd`); traffic on `<new>` > 0 |
| 3 Backfill | <repo>#<n> | Copy data, in batches | <how / not reversible: export at …> | Count = <expected>; dry-run count recorded |
| 4 Contract | <repo>#<n> | Remove `<old>` / shim | Restore the shim (never "restore data") | Zero hits on `<old>` for <period> |

Shims: <shim> — consumer <repo>, removal ticket <KEY>, metric <query>
Done when: step 4 merged and verified; no shim without a removal ticket.
```

A step is done when its "verify" is checked, not when its deploy turned green: CI proves the new
code works, not that the old path is unused.
