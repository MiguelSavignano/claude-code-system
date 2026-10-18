---
name: gitflow
description: The git flow for your team's web apps, using your reference repo as the model — a short-lived branch from `main`, a PR to `main` with unit tests and lint in CI, squash merge, and the merge deploys on its own — staging, e2e against staging, then production — with no manual step. Covers the commit convention (conventional commits, short and small), branch naming with the Jira key, and when a feature must go behind a feature flag to reach production safely. Use when starting a task, creating a branch, writing a commit, before merging a PR, or when someone asks "how does this reach production?". Invoke as /gitflow.
---

# gitflow — from a branch to production

**Type:** knowledge (executes nothing on its own).

Reference: **your reference repo** (`<reference-app>`: its `AGENTS.md` § Workflow and
`.github/workflows/deploy.yml`). The other web apps (`api`, `web`, `worker`) should follow the same
shape. This skill is the process; the per-pipeline facts live in `/ci-cd`.

## The flow

```
main ──► feat/PROJ-123-order-export ──► PR to main ──► CI (lint + unit) ──► review ──► squash merge
                                                                                          │
                       deploy.yml (push to main):  staging ──► e2e against staging ──► production
```

1. **Branch from an up-to-date `main`.** Short-lived: hours or a few days, not weeks.
   If a legacy `develop` branch exists but is abandoned, ignore it even if git metadata still lists
   it as the default branch (adapt to your repos).
2. **PR against `main`** — with `/pull-request`. While it is a draft, CI does not run (if your CI is
   configured that way).
3. **CI on the PR:** lint, unit tests and an image build. A red PR is not merged.
4. **Squash merge.** The PR title becomes the commit on `main` and GitHub appends `(#NNNN)`.
   Merge commits are disabled in the repo.
5. **The merge IS the deploy.** The push to `main` deploys staging, runs e2e against staging and,
   only if everything is green, deploys production. No manual step.

   If e2e fails, production is **not** deployed and `main` is ahead of production: fix it with
   another PR (fix forward) or revert the commit. Do not deploy by hand on top of it.

What runs at each step, which workflow deploys each app and from which branch, and how to trigger a
deploy by hand (a branch to staging to try it out, a hotfix during an incident): `/ci-cd`.

## Because the merge reaches production: feature flags

There is no intermediate step between "merged" and "in production". If the feature is **not ready
for users** (incomplete, pending validation in staging, depends on another repo that hasn't deployed
yet), merge it **behind a feature flag** that is off in production and turn it on when the time
comes. This way you integrate early, without long-lived branches, and deploy safely. When a flag is
needed: `/gate-policies`; how flags work in each repo: `/feature-flags`.

## Branches

`<type>/<JIRA-KEY>-<short-description>` — for example `feat/PROJ-123-order-export`,
`fix/PROJ-456-login-redirect`. Without a ticket: `chore/rubocop-todo`. The key in the branch and in
the PR title is what links the work to Jira.

## Commits

**Prefer conventional commits, short and small.** In English (per each repo's `AGENTS.md`).

```
type(scope): present-tense summary
```

- Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`, `perf`.
- `scope` = the area (`orders`, `auth`, `invoices`, `k8s`…).
- **Short:** the subject on one line, ~72 characters max, no trailing period. The *why* goes in the
  commit body or the PR, not in the subject.
- **Small:** one commit = one logical change that stands on its own. Separate refactors from
  behavior changes, and a migration from the code that uses it, when they can be reviewed apart.
- Jira key: if there is one, at the end of the subject — `feat(orders): close the free edit window (PROJ-123)`.

```
feat(orders): resolve the invoice template by customer type
fix(auth): keep the redirect after an expired session
test(orders): cover the template resolver without a customer
chore(k8s): disable the staging refresh
```

Because of squash, what stays on `main` is the **PR title**: it must follow the same convention.

## Rules

- Never push directly to `main`; everything goes in through a PR.
- Do not merge with red CI or with the PR in draft.
- Do not merge something you don't want in production today without a flag that turns it off.
- Before merging a PR with a migration, make sure the code deployed *before* the migration still
  works with it (staging and production don't change at the same time).
