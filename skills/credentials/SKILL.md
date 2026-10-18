---
name: credentials
description: Read AND write the secrets/configuration of an app in any environment, staging and production alike — which store it uses (sops-encrypted file, Rails encrypted credentials, or gh-secrets in GitHub Actions), plus non-secret config in `config/settings.yml`. Handles decrypting, editing, adding and rotating entries autonomously, without handing commands back to the user. Use when asked "how are secrets stored in X", "where's the credentials file", "decrypt/edit staging secrets", "add this secret", "configure production", or before touching any `.enc`/`.sops.yaml`/`credentials.yml.enc`/`settings.yml` file. Invoke as /credentials [app].
---

# Credentials & configuration — read and write, every environment

**Type:** procedure (reads **and writes** secrets) — the per-app facts live in your team wiki
(ideally reachable through a wiki MCP, if available).

**You operate these stores, you don't just describe them.** Decrypting, adding an entry, editing an
existing one, wiring a new environment — for staging *and* production — is your job to finish. See
[Autonomy](#autonomy) before deciding to hand anything back to the user; the answer is almost always
that you do it yourself.

## The four stores

Three hold **secrets**, one holds **non-secret configuration**. Which secret store an app uses is
per-app — **read it from the app's wiki note, don't assume** (a Rails app is *not* automatically a
Rails-credentials app; many use sops or gh-secrets instead).

| Store | Where the value lives | Committed? | Reviewable in a diff? |
|---|---|---|---|
| **`sops`** | age-encrypted file in the app's repo (`.env.*.enc`, `credentials/*.enc.env`, or an in-place encrypted YAML). Recipients (age **public** keys) declared in the repo's `.sops.yaml` | yes, encrypted | no |
| **`rails-credentials`** | `config/credentials.yml.enc` (shared) and/or `config/credentials/<env>.yml.enc` (per env), unlocked by `RAILS_MASTER_KEY` or `config/credentials/<env>.key`. Read in app code as `Rails.application.credentials.x.y` | yes, encrypted (keys are **not**) | no |
| **`gh-secrets`** | GitHub Actions repo/org secrets (`Settings → Secrets and variables → Actions`) and/or OIDC-assumed IAM roles. Nothing encrypted is committed; there is no file to decrypt | no | no |
| **`settings.yml`** | `config/settings.yml`, plaintext, `shared:` + one block per environment, loaded via `Rails.application.config_for(:settings)` and read as `Settings.x.y` | yes, plaintext | **yes** |

### Pick the store before you write anything

- **Is the value a genuine credential** — password, API key, token, signing secret, connection
  string with a password in it? → the app's secret store (`sops` / `rails-credentials` /
  `gh-secrets`, whichever the note says).
- **Is it a non-secret that merely differs per environment** — an internal URL, hostname, bucket
  name, feature flag, timeout, retry count? → **`config/settings.yml`. This is the default.**
  Encrypting a non-secret hides it from code review: nobody can see in a pull request that an API
  host changed. Do not put it in the secret store just because it varies by env.
- **Must it change without a deploy** (a kill switch)? → neither; that belongs in a runtime/DB
  store, and needs a design decision — raise it rather than encrypting it.

Only the secret stores are covered by this skill's encryption mechanics; `settings.yml` is an
ordinary tracked file you edit and commit like code.

Some repos carry Rails `config/credentials.yml.enc` scaffolding that was generated with the app and
never wired into deploy. A file's mere presence does not make it the live store — the wiki note's
`## Credentials` decides, and the repo's boot code / workflow is the tiebreaker.

### A plain `.env` is not a store — propose the migration

A bare `.env` / `.env.production` carrying real values is an **unmanaged** store: nothing is
encrypted, nothing is reviewable, and the values only exist on whichever machine happens to hold the
file. If you find the app configured that way, **stop and ask the user to migrate before you add
anything to it.** Do not extend a plain `.env` with a new secret — that entrenches the problem and
the value will not exist on the next deploy.

You are looking at an unmanaged `.env` when the app has a `.env`-family file with real values and
**none** of: an `.enc` sibling, a `.sops.yaml` in the repo, a `credentials.yml.enc` wired into boot,
or the deploy workflow reading `secrets.*`.

```bash
ls -a | grep -E '^\.env'          # .env, .env.production, .env.staging …
ls .sops.yaml config/credentials.yml.enc 2>/dev/null   # any managed store at all?
git ls-files | grep -E '\.env'    # worse: is a plaintext .env actually committed?
```

The migration to propose, split by what each value actually is:

1. **Secrets → `sops`.** Create/extend the repo's `.sops.yaml` with the team's age recipients, move
   the credential lines into an encrypted `credentials/<env>.enc.env` (or the app's in-place
   encrypted YAML), and point the deploy at it. The plaintext sibling goes in `.gitignore`.
2. **Non-secrets → `config/settings.yml`.** Internal URLs, hostnames, bucket names, flags, timeouts
   move into the `shared:` / per-environment blocks and are read as `Settings.x.y` — tracked in
   plaintext so changes show up in review.
3. **Nothing stays in `.env`** except local-development convenience values that are safe to
   regenerate and are already gitignored.

Say plainly what you found (which file, which values look like secrets vs config, whether it is
committed), then offer to carry out both halves. If a plaintext `.env` with live credentials is
committed to the repo, treat the secrets as **exposed** — removing the file does not unpublish it,
so flag that they need rotating too.

## How to answer

Read the app's note in your team wiki — the shared source of truth. If the wiki is served by an MCP
server, no local clone is needed; load its read/search/list tools first (if they are deferred, load
them all in one `ToolSearch` call).

Then, with `APP` = the argument, or the current repo's name when none was given:

1. Read the app's note (e.g. `Architecture/apps/<APP>.md` — adapt the path to your wiki layout).
2. If that 404s (an alias like `api` or `dashboard`), resolve the real note first: search the wiki
   with the app name, or list the app-notes folder to see every app note, then read the hit.
3. For the cross-app view — which apps use which strategy — read the wiki's deployed-apps overview
   (e.g. `Architecture/operations/deployed-apps.md`); the full SOPS+age procedure belongs in a
   runbook (e.g. `Runbooks/manage-credentials.md`).

**If the wiki is unreachable** (server down, MCP not authenticated, plugin not installed): say so and
stop rather than guessing a credentials path from memory — a wrong store or a wrong environment list
is worse than no answer. If it is an MCP, the user can authenticate with `/mcp` → *Authenticate*.

From the note read the **`## Credentials`** section (and the `credentials:` front-matter key):

- **strategy** — `sops`, `rails-credentials` or `gh-secrets` (front-matter `credentials:`).
- **file** — path(s) to the encrypted file, per env if it differs (e.g. `credentials/staging.enc.env`,
  `config/credentials/staging.yml.enc`); none for `gh-secrets`.
- **sops config** — the repo's `.sops.yaml` (only for `sops` apps).
- **environments** — which envs exist for this app (also derivable from its namespaces in the
  Monitoring section). You write **all** of them; see [Autonomy](#autonomy).
- the specifics: which secret names, how CI decrypts, any gotchas the note records.

When the note lacks a detail, the repo itself is ground truth — check its `.sops.yaml`,
`config/settings.yml`, `config/credentials/`, and workflow files.

If your organization does not keep per-app notes yet, use this template for each app's
`## Credentials` section. Fill this in for your organization:

| App | Strategy | File(s) | Environments | Notes |
|---|---|---|---|---|
| `<app>` | `sops` / `rails-credentials` / `gh-secrets` | e.g. `credentials/<env>.enc.env` | `staging`, `production` | how CI decrypts, gotchas |

## Per-store how-to

### `sops` apps

```bash
# Decrypt to stdout (never writes plaintext to disk)
sops --decrypt <file from the note>

# Edit in place — opens $EDITOR on the decrypted content, re-encrypts on save
sops <file from the note>

# Add a new recipient (new team member) — then re-encrypt every affected file
sops updatekeys <file>
```

The recipient list (age public keys) is in the repo's `.sops.yaml`, matched by `path_regex` against
the file path — check it if `sops` can't find a matching rule. Full walkthrough, including key
generation, revocation, and CI (`SOPS_AGE_KEY`) setup: your wiki's credentials runbook.

For a value-level write into an in-place encrypted YAML with per-env sections, prefer `sops set` over
opening an editor — it re-encrypts and regenerates the MAC. **Never hand-edit an `ENC[...]` file as
plaintext**; that corrupts the MAC and the file stops decrypting:

```bash
sops set config/application.yml '["staging"]["REPORTS_API_URL"]'    '"https://…"'   # value is JSON-encoded
sops set config/application.yml '["production"]["REPORTS_API_URL"]' '"https://…"'
```

**Never commit the decrypted plaintext.** The plaintext sibling (`.env`, `.env.production`, etc.)
must stay in `.gitignore`.

### `rails-credentials` apps

Rails' own encrypted store. The `.enc` file is committed; the key that opens it never is.

```bash
bin/rails credentials:show                              # shared config/credentials.yml.enc
bin/rails credentials:show --environment staging        # config/credentials/staging.yml.enc
```

Key resolution, in order: the `RAILS_MASTER_KEY` env var, else the key file —
`config/master.key` for the shared file, `config/credentials/<env>.key` for a per-env one. No key,
no read: `credentials:show` fails rather than returning partial content.

`credentials:edit` shells out to `$EDITOR` and re-encrypts on save, which needs an editor that
**blocks**. For an unattended write, point `EDITOR` at a script that mutates the decrypted YAML —
this is the reliable non-interactive path:

```bash
cat > /tmp/cred-edit <<'SH'
#!/bin/sh
printf '%s\n' "$NEW_ENTRY" >> "$1"      # $1 = the decrypted temp file
SH
chmod +x /tmp/cred-edit

export NEW_ENTRY='reports_api_url: https://…'
EDITOR=/tmp/cred-edit bin/rails credentials:edit --environment staging
```

Appending a **top-level** key is safe. To change a nested key, or one that already exists, read the
file first (`credentials:show`), rewrite the whole YAML in the script, and confirm with a second
`credentials:show` — a blind append under an existing parent produces a duplicate key that YAML
silently resolves to the last one.

Gotchas that bite:

- Editing with no key present **creates a new empty store** and re-encrypts it — silently discarding
  the previous content on that path. Always `credentials:show` first and confirm you can read it.
- A per-env file (`config/credentials/staging.yml.enc`) **shadows** the shared one for that
  environment. Adding a key to the shared file does nothing if a per-env file exists.
- The diff is unreviewable: the whole blob changes. `credentials:show` before and after is the only
  meaningful verification.
- The key files (`config/master.key`, `config/credentials/*.key`) must stay in `.gitignore`, and CI
  gets the value via `RAILS_MASTER_KEY` — check the deploy workflow actually sets it for the env you
  just wrote.

### `settings.yml` — non-secret configuration

An ordinary tracked file. Edit it, commit it, and it ships with the deploy — no keys, no encryption,
and the change is visible in review, which is the whole point.

```yaml
# config/settings.yml
shared:
  webhooks:
    max_consecutive_failures: 20
staging:
  reports_web:
    api_url: https://staging-reports.example.com
production:
  reports_web:
    api_url: https://reports.example.com
```

Read in code as `Settings.reports_web.api_url`, never as `ENV.fetch('REPORTS_WEB_API_URL',
'http://localhost:4000')` — a call-site default is the bug this file prevents. Add the key to
**every** environment block, not just the one you are working on; a missing production entry is a
production boot failure. `config_for` semantics, nesting and loading pitfalls are documented in the
`architect` agent's configuration section — follow it when you touch the loader.

### `gh-secrets` apps

There's no file — secrets are set directly on the repo (or org) and injected as env vars in the
GitHub Actions workflow:

```bash
gh secret list --repo <org>/<repo>              # see what's set (names only, no values)
gh secret set <NAME> --repo <org>/<repo>         # prompts for the value, or pipe it in
echo -n "$VALUE" | gh secret set <NAME> --repo <org>/<repo>
```

Cross-check the repo's deployment workflow (`.github/workflows/`) for exactly which
`secrets.*` / `vars.*` names it reads — the wiki note lists the ones already confirmed.
Some apps bundle a whole `.env` file into one secret (e.g.
`STAGING_ENV_VARIABLES`/`PRODUCTION_ENV_VARIABLES`) rather than one secret per key — check the note
before assuming a 1:1 mapping.

## Autonomy

**You configure every environment yourself — staging and production alike.** Do not ask permission
to touch an environment, and do not hand the user a command to paste. Enumerate the app's
environments from the wiki note and write **all** of them in the same change: a key that exists in
staging but not production is a production crash on the next deploy, and "remember to set this in
prod" is not a deliverable.

Do it without asking:

- Add a new key to any store, in any environment, including production.
- Read/decrypt any store in any environment to see what is there.
- Set, correct or complete a **non-secret** value anywhere — `settings.yml`, internal URLs,
  hostnames, flags, tuning numbers — using a value the user gave you or one you derived from ground
  truth you actually read (a k8s manifest, a Dockerfile, the service's own config). Say where a
  derived value came from.
- Backfill a key that is missing from one environment but present in another.
- Commit the resulting `.enc` / `settings.yml` change.

Three things still need the user, and none of them are "permission to configure production":

1. **A value only they have.** A vendor API key, a customer password, a third-party token — you
   cannot derive it and must not invent it. Ask for the value, then write it into every environment
   yourself. Asking for input you don't have is not asking for approval.
2. **Overwriting or rotating a secret that is already live in production.** Replacing a credential
   other systems currently authenticate with breaks them until they are updated, and the old value
   is unrecoverable once re-encrypted. Say exactly which secret and what will stop working, get a
   yes, then do it. Adding a *new* key is not this case; neither is anything in staging.
3. **Changing who can decrypt** — `sops updatekeys`, adding/removing an age recipient, rotating a
   data key or `RAILS_MASTER_KEY`. This affects other people's and CI's access, so confirm first.

An app on an unmanaged plain `.env` is the fourth: propose the migration instead of writing into it.

## Rules

- Don't guess a store from the stack (Rails ≠ automatically Rails credentials — many Rails
  apps use sops or gh-secrets instead). Always read the note's `## Credentials` first.
- Put the value in the **right** store: credentials in the secret store, everything else in
  `settings.yml`. Encrypting a non-secret hides it from review; leaving a real credential in
  plaintext exposes it.
- If a repo's actual files disagree with the note (new `.enc` file appeared, `.sops.yaml`
  changed), treat the live repo as ground truth and update the wiki note — it should say what was
  found and when.
- Never print decrypted secret values into a shared/committed context (chat is fine for the user
  who asked; never write plaintext into a file that could get committed).
- Verify after every write, per store: `sops --decrypt <file> | grep <KEY>`,
  `bin/rails credentials:show --environment <env>`, `gh secret list`. Then `git diff --stat` to
  confirm only the intended file changed and `git diff` to confirm no plaintext leaked in.

## Source of truth

The wiki note's `## Credentials` section is a **map**, not the secrets themselves — the actual
values only ever live in the encrypted file (decrypt locally), in Rails' encrypted credentials (needs
the key), or in GitHub's secret store (never readable, only settable). Non-secret configuration is
the exception: `config/settings.yml` *is* the value, in the open, on purpose. If the note looks
stale, re-derive from the repo's `.sops.yaml`, `config/credentials/`, `config/settings.yml` and
workflow files, then update the wiki note (not this skill).
