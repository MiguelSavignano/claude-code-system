---
name: skill-design
description: The rules for creating or reviewing a skill in this repo — one skill is the knowledge of one step, not a procedure; each fact lives in one skill and the others link to it; what the name and the description must say; facts with source and "Checked" date; the ~100-line limit; no org-specific details disguised as generic; safety rules — plus a review checklist and the closing steps (README skills table, links in other skills, PR). Use when someone says "create a skill", "add a skill for X", "review this skill" or "this should be a skill". Invoke as /skill-design [name].
argument-hint: "[name]"
---

# skill-design — the rules for a skill in this repo

Why small skills, and the lessons behind these rules: `docs/skill-design.md`.

## Where it goes

A clone of this repo, short-lived branch from `main`. Layout: `skills/<name>/SKILL.md`, plus
`references/` for detail that doesn't fit. Frontmatter: `name`, `description`, optionally
`argument-hint` and `allowed-tools`. Write in English.

## The rules

1. **One skill = the knowledge of one step**, not a procedure from start to finish. If the draft
   walks through a whole procedure, split it and point to the skills that already cover steps.
2. **A fact lives in one skill.** Before writing, `grep -rn "<term>" skills/`; if it's already
   there, link with `/name` instead of copying it.
3. **The name states the knowledge or the decision** (`secret-or-setting`, `feature-flags`), not a
   tool nor a generic verb. Area names already in use (`ci-cd`, `kubectl`) stay: renaming breaks links.
4. **The description says exactly what's in the body**, then "Use when…" and "Invoke as…". Claude
   picks the skill by it: trim the body, trim the description.
5. **Facts with source and date**: the file they come from (workflow, `CLAUDE.md`, wiki note) and
   "Checked YYYY-MM-DD". Unsourced → mark it "verify" or drop it. Never invent one.
6. **~100 lines.** Past that, split it or move detail to `references/`.
7. **No org-specific details disguised as generic.** Repo names, clusters, namespaces and stores go
   in a placeholder (see the README) or a "Fill this in for your organization" table.
8. **Safety.** Never print a secret's value. Confirm before anything in production and before any
   outward action (posting, merging, messaging, transitioning a ticket).

## Reviewing an existing skill

- [ ] One step, or a procedure? (rule 1)
- [ ] Any fact also in another skill? `grep -rn` it (rule 2)
- [ ] Does the description promise something the body no longer has, or miss what it has? (rule 4)
- [ ] Facts without source, or with a stale "Checked" date? (rule 5)
- [ ] Over ~100 lines? (rule 6)
- [ ] Org-specific values written as if they were universal? (rule 7)
- [ ] Prints secrets, or acts on production without confirming? (rule 8)

Propose the cut before rewriting.

## Closing

1. Add or update the skill's row in the README skills table (right section).
2. Update the skills that should link to it, and any link to a renamed or removed skill.
3. Open the PR with `/pull-request`.

If this repo ever becomes a plugin marketplace: bump the version in `plugin.json` and
`marketplace.json`, and run `claude plugin validate .` before the PR.
