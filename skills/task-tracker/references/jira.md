# Jira

Part of `/task-tracker`. Site `https://<org>.atlassian.net`; get the `cloudId` with
`getAccessibleAtlassianResources` and record it in your team wiki. Operate it with the Atlassian
MCP (`searchJiraIssuesUsingJql`, `getJiraIssue`, `createJiraIssue`, `editJiraIssue`,
`getTransitionsForJiraIssue`, `transitionJiraIssue`, `addCommentToJiraIssue`).

## Concepts

| Concept | In Jira |
|---|---|
| Project | A Jira project, key `PROJ`; tickets are `PROJ-123` |
| Ticket types | Native issue types. Investigation is a custom type: name it as your project does. Check with `getJiraProjectIssueTypesMetadata` |
| Epic → ticket → subtask | Native hierarchy (`parent` field) |
| States | Workflow statuses. Exact names with `getTransitionsForJiraIssue`; move with `transitionJiraIssue` |
| Sprints | Native sprints |
| Labels | Native labels |
| Blocking links | Issue links `blocks` / `is blocked by` (`createIssueLink`) |
| Criteria, artifacts, design | Paragraph custom fields (below) |
| Link to a PR | The key in the branch and PR title (GitHub for Jira app) plus a comment with the PR URL |

## Searches

| What | JQL |
|---|---|
| Open Epics | `project = PROJ AND issuetype = Epic AND statusCategory != Done ORDER BY created DESC` |
| Open sprint | `project = PROJ AND sprint in openSprints()` |
| In refinement | `project = PROJ AND status = Refining` |
| Missing criteria | `project = PROJ AND "Acceptance criteria" is EMPTY` |
| By text | `text ~ "invoice template" AND statusCategory != Done ORDER BY updated DESC` |

## Fields

Custom field keys differ per site: find them with `getJiraIssueTypeMetaWithFields` per issue
type, confirm on a test ticket, and fill in:

| Piece (`fields.md`) | Key | Type |
|---|---|---|
| Acceptance criteria | `customfield_<id>` | Paragraph |
| Output artifacts | `customfield_<id>` | Paragraph |
| UI design | `customfield_<id>` | Paragraph |
| Refinement checklist | `customfield_<id>` | Checkboxes (each option with its id) |

Write them with `editJiraIssue` and `contentFormat: "adf"`. In Markdown the MCP escapes the
checkboxes and they end up as literal text `\[ \]`.

```json
{"customfield_<id>": {"type": "doc", "version": 1, "content": [
  {"type": "taskList", "attrs": {"localId": "ac"}, "content": [
    {"type": "taskItem", "attrs": {"localId": "ac-1", "state": "TODO"},
     "content": [{"type": "text", "text": "Staging · … · …"}]}]}]}}
```

- Checkboxes: `taskList` → `taskItem` with `localId` and `state` `TODO` or `DONE`; inside,
  inline text (`code` and `link` marks are fine).
- Table: `table` → `tableRow` → `tableHeader` / `tableCell` → `paragraph`.
- When reading, `getJiraIssue` returns them as Markdown (`- [ ]`, tables) even if you ask for `adf`.

## Pitfalls

- Transitions are usually global: Jira does not stop a jump from `Refining` to `In Progress`.
- Put `Discard` in the Done category, and take the ticket out of the sprint first: Done counts
  its points as delivered.
