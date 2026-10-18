# Ticket fields

Part of `/task-tracker`. Every ticket carries the same pieces of spec, whatever the tool. Where
each one is stored depends on the tool: a dedicated field if the tool has rich-text custom
fields (Jira), or a fixed heading in the description if it does not (GitHub, Asana). The tool
reference says which, and how to write it so checkboxes stay checkboxes.

| Piece | Ticket types | What it holds |
|---|---|---|
| Description | all | Story, context (`/refinement-context`), scope and out of scope |
| Acceptance criteria | Story, Bug, Investigation | One checkbox per criterion (`/acceptance-criteria`) |
| Output artifacts | Story, Bug, Investigation | Table: artifact · where it lives · final state · criterion; the first row is "Where to test" (`/acceptance-criteria`) |
| UI design | Story | Only if something visible changes: link to the design canvas and what changes per screen (`/app-design`) |
| Refinement checklist | Story, Bug, Investigation | `Context`, `Acceptance criteria`, `Output artifacts` |

Fill in for your organization where each piece lives (field key, or heading name) and mark any
type you have not checked as "unchecked".

**When they go in the description**, use these exact headings so people and agents find them:

```markdown
## Acceptance criteria
- [ ] Staging · … · …

## Output artifacts
| Artifact | Where it lives | Final state | Criterion |
|---|---|---|---|
| **Where to test** | … | … | All |

## UI design
<canvas link> — <what changes per screen>
```

- The checklist is a signal, not evidence: `/definition-of-ready` looks at the content, not the boxes.
- **Beware of look-alike fields** (e.g. a `Design` field added by a design-tool integration):
  use only the field or heading you verified.
- Pending items and open questions go as comments, not in a field.
