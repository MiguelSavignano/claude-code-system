---
name: pull-request
description: Create a pull request in one of your team's repos with the team template (Why / What / Behaviour change / Tests), a conventional-commit title, and the ticket key linked if there is one (Jira, GitHub Issues, Asana or whatever tracker the team uses) — found in the branch, in the commits, or by searching the tracker — then leave a comment with the PR link on the ticket. Use when the user asks to "open/create a PR", "push this up", or when finishing a task on a branch. Invoke as /pull-request [TICKET-KEY] [--draft].
argument-hint: "[TICKET-KEY] [--draft]"
---

# pull-request — open a PR with the team template

**Type:** procedure (pushes the branch and creates the PR with `gh`).

General flow and commit convention: `/gitflow`. Reference example: pick one good PR in your
reference repo (`https://github.com/<org>/<reference-app>/pull/<n>`) and link it here.

## 1. Check the branch

```bash
git status --short
git rev-parse --abbrev-ref HEAD
git log --oneline origin/main..HEAD
```

- If you are on `main`, stop: create a branch first (`<type>/<KEY>-<description>`).
- If there are uncommitted changes, ask whether they belong in the PR.
- Base: `main` (read the trigger in the repo's `deploy.yml` if in doubt; some repos may still deploy
  from `develop`).

## 2. Find the ticket

Which tracker, its key format and how to search it: `/task-tracker`. In this order, and keep the
first one that turns up:

1. The argument (`/pull-request PROJ-123`, `/pull-request #123`).
2. The branch or the commits: `git rev-parse --abbrev-ref HEAD; git log --format=%s origin/main..HEAD`
   and search for your key format (e.g. `\b[A-Z][A-Z0-9]+-[0-9]+\b` for `PROJ-123`, `#[0-9]+`
   for GitHub issues).
3. If none, search the tracker by the text of the change, open tickets first, with the tool in
   its `/task-tracker` reference. Propose the candidate to the user; do not link it without
   confirmation.
4. If there is no ticket, the PR goes out without one. Do not create one on your own
   (`/task-tracker` if the user asks).

## 3. Title

Conventional commit, short, in English, with the key at the end if there is one. It will become the
commit on `main` after the squash.

```
feat(orders): resolve the invoice template by customer type (PROJ-123)
```

## 4. Body

Write it to a temporary file in the scratchpad, in English like the rest of the repo. Keep it short:
explain what the diff doesn't say. Delete the optional sections that don't apply.

```markdown
Ticket: [PROJ-123](<ticket URL>)

## Why

<The problem or need, in 2–5 lines. Concrete data if there is any.>

## What

- <Change 1: what and where (table, service, endpoint, screen).>
- <Change 2.>
- <Migrations, docs or OpenAPI updated.>

## Behaviour change after deploy

<Optional. What a user or integrator sees differently after deploy; a before/after table if it
helps. If it's behind a feature flag, say which one and its state in staging and production.>

## Tests

- `<command run>`: <actual result, e.g. 571 examples, 0 failures>.
- <What the new tests cover.>
- <Manual checks: migration up/down, staging, etc.>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

- The `Ticket:` line only if there is a ticket. With GitHub issues write `Refs #123`, never
  `Closes #123`: closing on merge would skip validation (`/task-tracker`).
- Under **Tests** put only what was actually run, with its result. If nothing was run, say so.
- If the repo has `.github/pull_request_template.md`, use it instead of this template.

## 5. Create the PR

```bash
git push -u origin HEAD
gh pr create --base main --title "<title>" --body-file <file> [--draft]
```

Return the PR URL to the user.

## 6. Link it on the ticket

With a ticket: comment on it with the PR URL (one line: `PR: <url> — <title>`), with the tool in
its `/task-tracker` reference. Tracker integrations may also link by the key in the branch and
title; the comment leaves a trail even if no integration is active. Do not change the ticket's
status unless the user asks.

## Rules

- Do not open the PR from `main`, or against a base other than the one the repo deploys from,
  without confirming.
- If the change must not be visible in production on merge, say so in the PR and name the flag
  (`/feature-flags`).
- Never invent test results or ticket keys.
