---
name: skill-design
description: How to create or review a skill in your team's skills repo / plugin marketplace — the rule that a skill is the knowledge of one step, not a whole procedure, that each fact lives in exactly one skill, the name, the description, sourced and dated facts, the ~100-line limit, which plugin it goes in, and the closing checklist (README, version in plugin.json and marketplace.json, `claude plugin validate .`, PR). Use when someone says "create a skill", "add a skill for X", "review this skill" or "this should be a skill". Invoke as /skill-design [name] [plugin].
argument-hint: "[name] [plugin]"
---

# Create or review a skill

Source: `docs/skill-design.md` (and `docs/contributing.md`, if your repo has one) in your team's
skills repo — the lessons learned from reorganizing skills are recorded there. Record when this was
last checked (e.g. "Checked YYYY-MM-DD").

> **One skill = the knowledge of one step, not a whole procedure.** Small skills combine
> (`/secret-or-setting` + `/feature-flags` + `/pull-request`) without repeating anything.

## Where to write it

Always in a clone of your team's skills repo (`github.com/<org>/<skills-repo>`), on a short-lived
branch from `main` (not in an app's repo, nor in `~/.claude/skills`). Choose the plugin by who needs
it. Fill this in for your organization:

| Plugin (`plugins/…`) | Goes here if… |
|---|---|
| `base` (`<org>`) | any team needs it and it doesn't operate deployed systems (Jira, docs, design, how to write skills) |
| `web` (`<org>-web`) | it's about the web apps and may touch staging or production |
| `<domain>` (`<org>-<domain>`) | it's specific to another team or domain (e.g. firmware, data, mobile) |

Layout: `plugins/<plugin>/skills/<name>/SKILL.md` (+ `references/`, `scripts/` if needed).
Frontmatter: `name`, `description`, optionally `argument-hint`, `allowed-tools`,
`disable-model-invocation: true` (purely procedural skills). Write in **English** so the skills
can be shared with contractors and other teams.

## Rules

1. **A fact lives in exactly one skill.** Before writing, `grep -rn` under `plugins/` for what you're
   about to say; if it's already there, link with `/name` instead of copying it.
2. **The name states the knowledge or the decision** (`secret-or-setting`, `feature-flags`), not the
   tool nor a generic verb.
3. **The description says what's in the body, and when to use it.** Claude picks the skill by its
   description: if you trim the body, trim the description.
4. **Facts with source and date**: the file they come from (workflow, `CLAUDE.md`, a team wiki
   note) and "Checked YYYY-MM-DD". Take them from the repos or the wiki, never from memory.
5. **Short.** If it goes past ~100 lines, it's probably two skills.

## Steps

1. State which step it covers. If it covers a whole procedure, propose how to split it and which
   existing skills already cover some of the steps.
2. Look for duplicates (rule 1) and pick the plugin (table above).
3. Write the `SKILL.md` with verified facts (rule 4).
4. Update the skills that should link to it and the plugin table in `README.md`.
5. Bump the plugin `version` in its `plugin.json` **and** in its entry in
   `.claude-plugin/marketplace.json` (if they don't match, the validator warns; `plugin.json` wins).
   If what the plugin contains changes, update its `description` in both places too.
6. `claude plugin validate .` and open the PR (`/pull-request`). After the merge, the team updates
   with `/plugin marketplace update <marketplace-name>`.

## Reviewing an existing skill

Same criteria: one step or a procedure? any fact repeated in another skill? does the description
promise something the body no longer has? facts without a source or with a stale date?
Propose the cut before rewriting.
