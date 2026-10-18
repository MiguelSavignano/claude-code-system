# Skill design: one skill = the knowledge of one step, not a whole procedure

An independent skill that knows **one step** (what is a secret and what is a setting, which
pipelines exist, how a PR is opened, how a feature flag works) is better than a skill that carries
a procedure from start to finish. Small skills combine into different scenarios without repeating
anything:

| Scenario | Skills that combine |
|---|---|
| Connect a new service | `/secret-or-setting` + `/feature-flags` + `/pull-request` |
| Feature that must not be visible in production yet | `/gitflow` + `/feature-flags` + `/ci-cd` |
| Hotfix during an incident | `/gitflow` + `/ci-cd` (manual deploy) + `/kubectl` |
| Investigate a production error | `/observability` (logs and traces) + `/kubectl` |

Lessons learned from reorganizing a web team's plugin (these failure modes repeat):

- `/deploy` carried the full procedure; it was cut down to a table and its description kept
  promising "launch, watch and report". On top of that, which branch deploys to which environment
  lived in three skills (`/deploy`, `/pipelines`, `/gitflow`) and each copy aged on its own. Now it
  lives only in `/ci-cd`.
- `/pipelines` was a 320-line scaffolding procedure that no longer described the real pipelines.
- `/env-vars` mixed the important decision (secret or setting?) with how to write to four stores;
  the decision got buried. Now it's `/secret-or-setting`, and the detailed writing is in
  `/credentials`.
- `/app-logging` and `/logs` were the write side and the read side of the same thing; `/logs` was
  deleted and `/app-logging` kept linking to it. Now `/observability` covers logs and traces, and
  reading uses a Loki query tool / MCP, if available.

Rules:

1. **A fact lives in exactly one skill.** The others link to it with `/name`; they don't copy it.
2. **The name states the knowledge or the decision** (`secret-or-setting`, `feature-flags`), not the
   tool nor a generic verb.
3. **The description says what's in the body.** Claude picks the skill by its description: if you
   trim the body, trim the description.
4. **Facts with source and date**: the file they come from (workflow, `AGENTS.md`, wiki note) and
   "Checked YYYY-MM-DD".
5. **Short.** If it goes past ~100 lines, it's probably two skills.

To create or review a skill with Claude Code, use the `/skill-design` skill from your base plugin:
it applies these rules, picks the plugin, and closes with version bump, `claude plugin validate .`
and PR.
