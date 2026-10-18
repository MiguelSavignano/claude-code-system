# Asana

Part of `/task-tracker`. Tickets are tasks in one Asana project. Operate it with an Asana MCP if
connected (authenticate, then list its tools: names vary by server) or the REST API with a
personal access token. Record in your team wiki: the workspace and project gids, the section
gids and the custom field gids.

## Concepts

| Concept | In Asana |
|---|---|
| Project | One Asana project for all new work |
| Ticket key | The task URL, or a custom ID field (e.g. `PROJ-123`) if your plan has it: then use it in branches and PR titles |
| Ticket types | A single-select custom field `Type` (Epic, Story, Task, Bug, Investigation) |
| Epic → ticket → subtask | Fill in: the Epic as a parent task with the Stories as subtasks (also add them to the project, or the board does not show them), or one project per Epic in a portfolio |
| States | Board sections named exactly as the states of `/task-tracker`. Mark the task complete only in `Done` or `Discard` |
| Sprints | No native sprints: a single-select custom field `Sprint`, or one project per sprint |
| Labels | Tags, or a multi-select custom field (easier to filter and to keep to a fixed list) |
| Blocking links | Native dependencies ("blocked by") |
| Criteria, artifacts, design | Headings in the description (`fields.md`) |
| Comments | Comments ("stories" in the API) |
| Link to a PR | A comment with the PR URL; the key in the branch and PR title if you use custom IDs |

## Writing the description

Asana rich text has lists but **no checkboxes and no tables**. Keep the `fields.md` headings and:

- Criteria: one line per criterion starting with `[ ]` / `[x]` as plain text. Do not turn them
  into subtasks: subtasks are the work (one per PR), criteria are the check.
- Artifacts: one line per artifact, `<artifact> · <where it lives> · <final state> · <criterion>`,
  the first one `Where to test · …`.

## Pitfalls

- Subtasks not added to the project are invisible on the board and in project searches.
- If the GitHub integration is enabled, check whether it completes tasks when a PR merges:
  merged is not delivered, and that would skip `Validating`.
- Completed tasks are hidden by default in searches: include them when looking for previous work.
