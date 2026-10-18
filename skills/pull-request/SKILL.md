---
name: pull-request
description: Create a pull request in one of your team's repos with the team template (Why / What / Behaviour change / Tests), a conventional-commit title, and the Jira key linked if there is one — found in the branch, in the commits, or via an Atlassian/Jira MCP search — then leave a comment with the PR link on the ticket. Use when the user asks to "open/create a PR", "push this up", or when finishing a task on a branch. Invoke as /pull-request [JIRA-KEY] [--draft].
argument-hint: "[JIRA-KEY] [--draft]"
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

## 2. Find the Jira ticket

In this order, and keep the first one that turns up:

1. The argument (`/pull-request PROJ-123`).
2. The branch or the commits: `git rev-parse --abbrev-ref HEAD; git log --format=%s origin/main..HEAD`
   and search for `\b[A-Z][A-Z0-9]+-[0-9]+\b` (your project keys, e.g. `PROJ-123`).
3. If none, search with an Atlassian/Jira MCP if available (`searchJiraIssuesUsingJql`, site
   `<org>.atlassian.net`) using the text of the change, e.g.
   `text ~ "invoice template" AND statusCategory != Done ORDER BY updated DESC`. Propose the
   candidate to the user; do not link it without confirmation.
4. If there is no ticket, the PR goes out without one. Do not create one on your own (`/jira-task`
   if the user asks).

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
Jira: [PROJ-123](https://<org>.atlassian.net/browse/PROJ-123)

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

- The `Jira:` line only if there is a ticket.
- Under **Tests** put only what was actually run, with its result. If nothing was run, say so.
- If the repo has `.github/pull_request_template.md`, use it instead of this template.

## 5. Create the PR

```bash
git push -u origin HEAD
gh pr create --base main --title "<title>" --body-file <file> [--draft]
```

Return the PR URL to the user.

## 6. Link it in Jira

With a ticket: `addCommentToJiraIssue` with the PR URL (one line: `PR: <url> — <title>`). The
GitHub-for-Jira integration also links by the key in the branch and title; the comment leaves a
trail even if that integration isn't active. Do not change the ticket's status unless the user asks.

## Rules

- Do not open the PR from `main`, or against a base other than the one the repo deploys from,
  without confirming.
- If the change must not be visible in production on merge, say so in the PR and name the flag
  (`/feature-flags`).
- Never invent test results or Jira keys.
