# GitHub Issues + Projects

Part of `/task-tracker`. Tickets are issues; the board is a GitHub Project (`github.com/orgs/<org>/projects/<n>`).
Operate it with `gh`. Project commands need the `project` scope: `gh auth refresh -s project`.
Record in your team wiki: the project number, its node id (`gh project view <n> --owner <org> --format json`)
and the Status and Iteration field and option ids (`gh project field-list <n> --owner <org> --format json`).

## Concepts

| Concept | In GitHub |
|---|---|
| Project | One org Project; issues can live in several repos. Fill in which repo new tickets go to (often a dedicated planning repo) |
| Ticket key | `#123` in the same repo, `<org>/<repo>#123` across repos |
| Ticket types | Org issue types (Task, Bug, Feature by default; an admin can add Epic, Investigation). Without them, use labels `type:epic`, `type:investigation`… |
| Epic → ticket → subtask | Sub-issues: the Epic is a parent issue, Stories/Tasks its sub-issues, subtasks theirs |
| States | The Project's **Status** single-select field, with the options of `/task-tracker`. Open/closed is separate: close the issue only on `Done` (`completed`) or `Discard` (`not planned`) |
| Sprints | The Project's **Iteration** field |
| Labels | Native labels (repo-level: create the policy labels in every repo that holds tickets) |
| Blocking links | Issue dependencies ("blocked by"), if enabled in your org; otherwise a `Blocked by #123` line in the body |
| Criteria, artifacts, design | Headings in the issue body (`fields.md`): Project custom fields cannot hold checklists or tables |
| Link to a PR | `Refs #123` in the PR body: it links without closing |

## Commands

```bash
gh issue create --repo <org>/<repo> --title "…" --body-file <file> --label "…"
gh project item-add <n> --owner <org> --url <issue-url>          # put it on the board
gh issue view 123 --repo <org>/<repo> --json title,body,labels,state,assignees,comments
gh project item-list <n> --owner <org> --format json --limit 500  # status per item: filter with jq
gh project item-edit --project-id <project-id> --id <item-id> \
  --field-id <status-field-id> --single-select-option-id <option-id>   # move state
gh issue comment 123 --repo <org>/<repo> --body "…"
gh issue close 123 --repo <org>/<repo> --reason "not planned"    # Discard
gh search issues "org:<org> invoice template" --state open        # search by text
```

Sub-issues and dependencies are not in every `gh` version: use the REST API
(`gh api repos/<org>/<repo>/issues/<n>/sub_issues -F sub_issue_id=<issue id, not number>`) and
check the current docs before relying on it.

## Pitfalls

- **`Closes #123` / `Fixes #123` in a PR closes the issue on merge**, and the Project's default
  workflows then move it to Done. Merged is not delivered: write `Refs #123`, and disable the
  "Pull request merged" workflow. The "Item closed" workflow also sets Status to Done when you
  close a ticket as Discard: set Status to `Discard` after closing, or disable it.
- Issue body edits are versioned (edit history), but the history is easy to miss: leave a
  comment when criteria change.
- Labels are per repo: a label missing in one repo silently splits the policy.
