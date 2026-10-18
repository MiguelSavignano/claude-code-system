---
name: rollout-plan
description: Plan a breaking or destructive change — an API contract change, a schema/column change, removing a used code path, a cross-repo migration — as a sequenced, verifiable rollout instead of an edit-and-deploy. Use before removing/renaming an endpoint, column, queue, topic, or config key that something else might depend on; before any migration that alters or drops data; or when asked to "refactor", "clean up", "remove the old X", "migrate off Y", or "deprecate Z". Invoke as /rollout-plan [what you're changing].
---

# rollout-plan — sequence breaking changes, don't wing them

**Type:** method (planning) — produces a sequenced plan, executes nothing.

**The rule this skill enforces:** a breaking change is not one commit, it is a sequence of
deploys, and the sequence is the design. Deleting the old thing in the same change that
introduces the new one is not a refactor, it's a bet that nothing else was calling the old
thing — and "I checked this repo" is not the same bet as "I checked every repo." When the bet
is wrong, the fix under pressure is always worse than the plan would have been: a compatibility
shim bolted on after the fact, with no owner and no removal date, which is how "temporary" code
becomes permanent.

## The motivating case (generalized — this pattern repeats)

An API endpoint in one service got replaced with a better-designed one in the same PR that
deleted the old one. The old endpoint looked unused from inside that repo: no code in that
codebase called it anymore. It was not unused — a *different* repo had a merged, deployed,
actively-calling client built against it, discovered only by grepping sibling repos and checking
real deploy history (`gh run list`), not by reading the one repo being changed. The fix was a
deprecated alias added under time pressure, mid-review, instead of as part of a plan — exactly
the "temporary shim with no removal date" this skill exists to prevent. Read the code you're
deleting for callers; that only proves callers *in this repo*. A contract has callers wherever
its consumers live, and none of them show up in `grep -r` from where you're standing.

## Step 1 — Map the blast radius before touching anything

Before removing or changing behavior of anything another system might depend on:

- **Grep every sibling repo you have local access to**, not just the one you're editing — for
  the literal path/column/queue name/config key, and for the client class that would call it
  (`<App>::Client`-shaped names, `*Client`, `*_service.rb`).
- **Check your team wiki's per-app notes** (architecture / integration pages) and any
  proposal/runbook docs for the feature — they record cross-service integrations that a single
  repo's code never mentions, including *why* a contract exists and who calls it.
- **Check real deploy history, not just merged PRs**: `gh pr view <n> --repo <org>/<repo>
  --json state,mergedAt` tells you code exists; `gh run list --repo <org>/<repo>` (filter for the
  deploy workflow) tells you whether it's actually running somewhere. A merged-but-undeployed PR
  is a different risk level than one that's been live in production for a month.
- **State what you found before proposing removal.** "Nothing in this repo calls it" is not the
  same claim as "nothing calls it" — say which one you're making.

If you find a live caller you're about to break, stop and surface it before writing the removal
— that's a decision for whoever owns the timeline, not something to route around silently by
either abandoning the refactor or shipping the breakage.

## Step 2 — Default to expand → migrate → contract

Three separate deploys, not one:

1. **Expand.** Add the new contract (endpoint, column, queue) *alongside* the old one. Nothing
   that currently works stops working. Ship this alone; it is backward compatible by
   construction, so it carries the least risk and can go out anytime.
2. **Migrate.** Update every known consumer, one at a time if there are several, to use the new
   contract. Deploy each, then **verify it's actually using the new path in that environment**
   before moving to the next consumer or the next environment — see Step 4. This is the phase
   that takes calendar time; resist the urge to skip straight to Step 3 because the code change
   itself is small.
3. **Contract.** Only after every consumer is confirmed migrated in every environment that
   matters, remove the old contract. This is a separate deploy from the last migration step, not
   the same one — so if removal breaks something, the blast radius of the rollback is just the
   removal.

Collapsing these into one deploy is the exact mistake this skill exists to prevent: it deletes
your ability to roll back independently of whether every consumer actually migrated.

## Step 3 — If you need a shim anyway, give it a death, not just a birth

Sometimes Step 1 finds a live consumer *after* the new contract already shipped, or the old and
new contract must coexist for longer than one deploy cycle for reasons outside your control. A
shim (deprecated alias route, dual-write column, compatibility flag) is a legitimate tool here —
but only if it ships with:

- **A named reason it exists** — which specific consumer, which specific PR/version, is still on
  the old contract. Not "for backward compatibility" — that's not falsifiable.
- **A removal trigger you can actually check**, not a vague "later": a linked follow-up
  issue/PR in the *other* repo, or a log line / metric on hits to the old path so you can prove
  zero traffic before deleting it instead of guessing. "Nobody's complained" is not evidence of
  zero traffic; an unused deprecated path can run silently forever.
- **A comment at the shim itself** naming the consumer and pointing at the follow-up, so the
  next person who finds it doesn't have to re-derive why it's there (e.g. a `# Source: <consumer
  repo/PR> — remove after <follow-up issue>` comment directly above the alias route or column).

A shim with all three of these is a planned, temporary bridge. A shim with none of them is
permanent code wearing a "deprecated" label that nobody will ever act on.

## Step 4 — Verify at each phase, in the environment, not just in CI

"Tests pass" proves the new code isn't broken. It does not prove the old code is unused, and it
does not prove a consumer in another repo actually adopted the new contract after its deploy.
Before moving from one phase to the next:

- Confirm the consumer's deploy actually happened (`gh run list` for its deploy workflow, not
  just that its PR merged — a merge doesn't deploy itself if the workflow is manual-dispatch).
- Confirm real traffic is hitting the new path where possible (`/observability`, a Loki count query for the
  new endpoint's request count > 0, or an application-level counter) rather than inferring it
  from "the deploy succeeded."
- Only then treat that phase as done. A phase that "should" be done but hasn't been checked is
  not done — it's a claim.

## Database and data-affecting changes — the same three phases, plus:

Schema and data changes follow the identical expand → migrate → contract shape, with the
consequence of skipping it being unrecoverable instead of just an outage:

- **Back up before any migration that alters or drops data**, even in staging if the data has
  any value to anyone — use the app's own backup mechanism if one exists (e.g. a team `/db-backup`
  skill or documented backup job) rather than improvising a `pg_dump` invocation.
  "It's just staging" is not a reason to skip this; staging data gets used for real testing and
  demos, and regenerating it can cost more than the backup would have.
- **Prefer reversible migrations** (Rails `change` blocks Rails can invert, or explicit `up`/
  `down`). If a migration genuinely can't be reversible, say so explicitly in the PR description
  — don't let that be a silent property someone discovers during an incident.
- **Renaming or dropping a column is three migrations, not one**: add the new column and
  backfill it → update every reader/writer to use the new column (dual-write if reads and writes
  are split across deploys) → drop the old column in a *later* migration, after confirming
  nothing still reads it. Dropping a column in the same migration that starts writing the new one
  removes your rollback path the moment something you missed still reads the old one.
- **Dry-run destructive queries first.** Before a migration or one-off script that updates or
  deletes many rows, run the equivalent `SELECT COUNT(*)` with the same `WHERE` clause and look at
  the number. A count that's 100x what you expected is cheaper to catch before the `DELETE` than
  after.
- **Large backfills run in batches**, not as one long-held transaction/lock — especially on a
  table anything else reads from during business hours.

## Step 5 — Write the sequence down before executing it

Even a short plan, in the PR description or a scratch note, beats carrying the sequence in your
head:

- Which repos change, in what order, and why that order (usually: provider's expand phase before
  any consumer's migrate phase; every consumer's migrate phase, verified live, before the
  provider's contract phase).
- What "verified" means at each step — the specific log line, dashboard, or check that proves it,
  not just "deploy succeeded."
- What the rollback is at each step if something goes wrong, and how far back it reaches (a
  contract-phase rollback should only ever mean "restore the shim," never "restore deleted data").
- What "done" looks like for the whole sequence — the artifact that lets someone (including
  future-you) close it out instead of leaving a shim to rot.

This is what turns "we'll remove the alias later" into "we remove the alias on this date, because
the plan said so and the traffic metric confirmed it" — the difference between a plan and a hope.

## Rules of thumb

- A compatibility shim without a named owner and a checkable removal trigger is not temporary —
  budget for it as permanent, because that's what it will become.
- Grepping your own repo proves nothing about callers that live elsewhere. Check sibling repos,
  the team wiki, and actual deploy history before you believe "nothing depends on this."
- One deploy that both adds the new and removes the old is a bet with no rollback. Split it into
  at least two, with a verification step between them.
- "CI is green" answers "is the new code broken?" It does not answer "is the old code still
  needed?" — those are different questions with different evidence.
- For data: backup first, dry-run destructive counts first, and never drop a column or table in
  the same release that stops writing to it.
- If you can't check whether a shim is still receiving traffic, that's a gap to close (add the
  metric/log line) before you can responsibly plan its removal — not a reason to remove it blind
  or leave it forever.
