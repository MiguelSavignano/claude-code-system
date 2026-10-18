---
name: pull-request
description: Open a pull request in one of your team's repos — find the ticket key (argument, branch, commits, then ask; tracker search only if a tool is available), write the body with the repo's `.github/pull_request_template.md` if it exists or else the team template (Why / What / Behaviour change / Tests), link the ticket with a non-closing `Refs <KEY>`, choose draft or ready, and — after showing the user the title, body and ticket and getting confirmation — push the branch, run `gh pr create` against the repo's deploy branch and comment the PR link on the ticket. Use when the user asks to "open/create a PR", "push this up", or when finishing a task on a branch. Invoke as /pull-request [TICKET-KEY] [--draft].
argument-hint: "[TICKET-KEY] [--draft]"
---

# pull-request — open a PR with the team template

**Type:** procedure (pushes, creates the PR with `gh`, comments on the ticket). Branch, commit and
title convention: `/gitflow` · deploy branch per repo: `/ci-cd` · key format, search, comment: `/task-tracker`.

## 1. Check the branch

```bash
git fetch origin
git status --short
git log --oneline origin/<base>..HEAD    # <base> = the branch this repo deploys from (/ci-cd)
```

On `<base>` itself, stop and branch first (`/gitflow`). Uncommitted changes: ask whether they belong.

## 2. Find the ticket key

First hit wins: the argument → the branch name → commit subjects (`git log --format=%s
origin/<base>..HEAD`) → ask the user. Search the tracker by text only if a tracker tool is
available, to propose a candidate. A key not given as the argument is **confirmed with the user**
before use. No ticket is fine: the PR goes out without one; never create or invent one.

## 3. Title and body

- **Title:** follows the commit convention in `/gitflow` (it becomes the commit on merge).
- **Body:** if the repo has `.github/pull_request_template.md`, use it — it wins. Otherwise:

```markdown
Refs <KEY>

## Why
<The problem or need in 2–5 lines; concrete data if any.>

## What
- <Each change: what and where (table, service, endpoint, screen); migrations, docs.>

## Behaviour change after deploy
<Optional. What users or integrators see differently; flag name and state if behind one.>

## Tests
- `<command run>`: <actual result>. <Manual checks.> If nothing was run, say so.
```

- **`Refs <KEY>`, never `Closes` / `Fixes`:** the tracker state is moved by people, and a closing
  keyword moves it on merge, before validation (`/task-tracker`). Omit the line with no ticket.
- Explain what the diff does not; drop optional sections that don't apply; never invent results.
  Write it to a scratchpad file. Add a footer/attribution only if the user's or harness's policy asks.

## 4. Draft or ready

**Draft** if the user passes `--draft`, the work is incomplete or CI should not run yet; otherwise
ready. If it must not be visible in production on merge, name the flag that hides it (`/feature-flags`).

## 5. Confirm, then push and create

Show the user the title, base, body, draft/ready and the target ticket; only after an explicit yes:

```bash
git push -u origin HEAD
gh pr create --base <base> --title "<title>" --body-file <file> [--draft]
```

Return the PR URL. With a ticket, comment `PR: <url> — <title>` on it (how: `/task-tracker`); it is
outward-facing too, so include it in that confirmation. Do not change the ticket's state unless asked.
