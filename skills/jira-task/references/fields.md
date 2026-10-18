# Ticket fields in PROJ

Part of `/jira-task`. Custom field keys differ per Jira site: find yours with
`getJiraIssueTypeMetaWithFields` for each issue type, write a field on a test ticket to confirm,
and fill in the table below. Mark any type you have not checked as "unchecked".

| Field | Key | Type | Issue types | What it holds |
|---|---|---|---|---|
| Description | `description` | system | all | Story, context (`/refinement-context`), scope and out of scope |
| Acceptance criteria | `customfield_<id>` | Paragraph | Story, Bug, Investigation | Native checkboxes, one per criterion (`/acceptance-criteria`) |
| Output artifacts | `customfield_<id>` | Paragraph | Story, Bug, Investigation | Table: artifact · where it lives · final state · criterion; the first row is "Where to test" (`/acceptance-criteria`) |
| UI design | `customfield_<id>` | Paragraph | Story | Only if something visible changes: link to the design canvas and what changes per screen (`/app-design`) |
| Refinement checklist | `customfield_<id>` | Checkboxes | Story, Bug, Investigation | Options `Context`, `Acceptance criteria`, `Output artifacts` (each with its option id) |

- The checklist is a signal, not evidence: `/definition-of-ready` looks at the fields, not the boxes.
- **Beware of look-alike fields** (e.g. a `Design` field added by a design-tool integration):
  use only the field you verified above.
- Pending items and open questions go as comments, not in a field.

## How to write them

With `editJiraIssue` and `contentFormat: "adf"`. In Markdown the MCP escapes the checkboxes and
they end up as literal text `\[ \]`.

```json
{"customfield_<id>": {"type": "doc", "version": 1, "content": [
  {"type": "taskList", "attrs": {"localId": "ac"}, "content": [
    {"type": "taskItem", "attrs": {"localId": "ac-1", "state": "TODO"},
     "content": [{"type": "text", "text": "Staging · … · …"}]}]}]}}
```

- Checkboxes: `taskList` → `taskItem` with `localId` and `state` `TODO` or `DONE`; inside,
  inline text (the `code` and `link` marks are fine).
- Table: `table` → `tableRow` → `tableHeader` / `tableCell` → `paragraph`.
- When reading them, `getJiraIssue` returns them as Markdown (`- [ ]`, `- [x]`, tables) even if
  you ask for `adf`.
- To find tickets missing them: `project = PROJ AND "Acceptance criteria" is EMPTY` (same with
  `"Output artifacts"`).
