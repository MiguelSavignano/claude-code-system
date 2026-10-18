---
name: gitflow
description: The branch and commit flow for your team's repos — trunk-based with short-lived branches, not the classic git-flow model — a short branch from the repo's base branch named with the ticket key, conventional commits (format, short and small, in English), one subtask = one branch = one PR, rebase on the base branch before merging, squash merge so the PR title becomes the commit, and never pushing to the base branch directly. Links to /ci-cd for deploys, /pull-request for the PR, and /gate-policies, /feature-flags and /rollout-plan for flags and migrations. Use when starting a task, creating a branch, writing a commit or before merging a PR. Invoke as /gitflow.
---

# gitflow — branches and commits

**Type:** knowledge (executes nothing on its own). Trunk-based development with short-lived
branches, **not** the classic git-flow model (no `develop`/`release`/`hotfix` branches); the name
is kept so existing links don't break.

```
<base> ──► feat/PROJ-123-order-export ──► PR ──► rebase on <base> ──► squash merge into <base>
```

The **base branch is per repo** (`main` in most, not all): which one, what runs on the PR and what
the merge deploys live in `/ci-cd`. Merging deploys; where and how: `/ci-cd`.

## Branches

1. **Branch from an up-to-date base branch.** Short-lived: hours or a few days, not weeks.
2. **One subtask = one branch = one PR.** Two subtasks are two branches, even if small.
3. **Name:** `<type>/<TICKET-KEY>-<short-description>` — `feat/PROJ-123-order-export`,
   `fix/PROJ-456-login-redirect` (GitHub issues: `feat/123-order-export`). Without a ticket:
   `docs/update-setup-guide`. The key in the branch and in the PR title links the work to the
   tracker (key format: `/task-tracker`).
4. **Before merging, update with the base:** `git fetch origin && git rebase origin/<base>`, then
   `git push --force-with-lease` (only on your own branch). Re-check CI after the rebase.
5. **Squash merge.** The PR title becomes the single commit on the base branch, so it follows the
   commit convention below. Which merge methods are allowed: check the repo's settings.

The PR itself (body, ticket link, draft vs ready): `/pull-request`.

## Commits

```
type(scope): present-tense summary
```

- Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`, `perf`.
- `scope` = the area (`orders`, `auth`, `invoices`…). English, per each repo's `AGENTS.md`.
- **Short:** subject on one line, ~72 characters max, no trailing period. The *why* goes in the
  commit body or the PR.
- **Small:** one commit = one logical change that stands on its own. Separate refactors from
  behavior changes when they can be reviewed apart.
- Ticket key, if there is one, at the end of the subject: `feat(orders): export orders as CSV (PROJ-123)`.

```
feat(orders): resolve the invoice template by customer type
fix(auth): keep the redirect after an expired session
test(orders): cover the template resolver without a customer
```

## Rules

- Never push directly to the base branch; everything goes in through a PR.
- Do not merge with red CI.
- Not ready for users but merging anyway? Whether it needs a flag: `/gate-policies`; how:
  `/feature-flags`.
- Migrations and cross-repo contracts, and how to split them into PRs: `/rollout-plan`.
