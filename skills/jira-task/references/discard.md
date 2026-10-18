# Discarding a ticket

Part of `/jira-task`. The state is called `Discard` (category Done).

Discarding is the owner's decision, never an agent's, and it leaves a comment: the reason and who decided.

- **Nothing was installed** (in `To Do`, `Refining` or `Refined`) → `Discard` directly.
- **Something is installed** (merged PR, flag, migration, config) → do not discard it outright:
  move it back to `Refining`, the spec becomes "remove what was installed", and the rollback is
  refined, deployed and validated like any other change. It enters `Discard` once it is verified
  that nothing is left.

Do not rewrite the criteria: strike them through or mark them inline ("discarded", "rolled back");
Jira keeps the description history.

```markdown
- [ ] ~~Production · the report uses the new endpoint~~ — discarded
- [x] Staging · flag `new_report` enabled — rolled back
- [ ] Production · the flag `new_report` no longer exists in code or config (rollback)
```

Example: it is installed behind a flag, in `Validating` it turns out not to be safe → `Refining`
with the reason → AC to remove the flag and the code → `Refined` → `In Progress` → `Validating`
(verified the flag no longer exists) → `Discard`. Before discarding a ticket inside a sprint,
take it out of the sprint: the "Done" category counts its points as delivered.
