---
name: secret-or-setting
description: Classify a new configuration value for an app — is it a secret (sops), a setting (`config/settings.yml`), or a runtime switch? — and write it to EVERY deployed environment, staging and production, in the same change as the code that reads it. Looks up each app's store in your team wiki (via a wiki MCP, if available). Prevents the classic bug: `ENV.fetch('X', 'http://localhost…')` in code with no value in the environments, or a hostname encrypted in sops. Use when adding or renaming an environment variable or setting, wiring up a new service, or when asked "add this ENV", "configure staging/production". Invoke as /secret-or-setting [app] [NAME].
argument-hint: "[app] [NAME]"
---

# secret-or-setting — classify and write a configuration value

**Type:** procedure (writes the configuration of every environment). Writing companion to
`/credentials`, which has the per-store detail and the autonomy policy.

**The rule:** as soon as code reads a value at runtime, that value is a deploy requirement. The
change is not done until the value exists in **every** deployed environment. Otherwise the next
deploy boots without it: the pod crash-loops or, worse, boots "healthy" pointing at localhost.

## 1. Classify — the most important step

Ask: **does it grant access to something?** If you posted it in a public channel, could someone log
in, sign, or impersonate you?

| It is… | Examples | Goes in | Read as |
|---|---|---|---|
| **Secret** — grants access | passwords, API keys, tokens, JWT/signing secrets, private keys, `client_secret` | `credentials/<env>.enc.env` (sops) | environment variable; in Rails, via an ERB tag in `settings.yml` |
| **Setting** — grants no access, varies by environment | service URLs, hostnames, buckets, region, tables, mailboxes, timeouts, limits, simple flags, `LOG_LEVEL`, `client_id` | `config/settings.yml` (Rails) / the app's equivalent settings file (e.g. `config/settings.yaml` in a Python app), plaintext and committed | `Settings.x.y` / `settings.X` |
| **Runtime switch** — changes without a deploy | kill switches, incident thresholds, per-user flags | a DB-backed app configuration table or a feature flag (`/feature-flags`) | — |

- **Default to setting.** A hostname or a table name is not a credential; in `settings.yml` the
  deployed value is reviewed in the PR diff. Encrypting it hides it without protecting it.
- **Mixed value** (`postgres://user:pass@host/db`): split it — host and database in the setting,
  password in the secret. If it can't be split, the whole thing is a secret.
- **When in doubt, secret.** Doubt changes the store, not who does the work.

Example: a backend API calling an internal reports service. `reports_service_url`
(`http://reports.<app>-production.svc.cluster.local`) is a **setting** in `config/settings.yml`;
`REPORTS_SERVICE_TOKEN` is a **secret** in sops, referenced from `settings.yml` with
`<%= ENV.fetch('REPORTS_SERVICE_TOKEN', '') %>` and enforced in `REQUIRED_KEYS`.

**Never:**
- `ENV.fetch('X', 'http://localhost:…')` in code. A local default does not rescue staging or
  production. The value goes in the store and is read from `Settings`.
- A plaintext `.env` as the store for a deployed environment. If the app only has that, stop and
  propose migrating (secrets to sops, the rest to `settings.yml`) — `/credentials`.

## 2. Find the app's store and environments — team wiki

Each app's note in your team wiki says which store it uses and which namespaces it deploys to. Read
it through the wiki MCP, if available (load its read/search tools first if they are deferred):

```
wiki read  → Architecture/apps/<app>.md     (adapt to your wiki layout; if 404, search by app name)
```

- `## Credentials`: strategy (sops / rails-credentials / gh-secrets) and files.
- `## Monitoring`: namespaces `<app>-<env>` → **that is the list of environments; write all of them**.

If the wiki tools don't show up, the MCP isn't authenticated: ask the user to run `/mcp` → *<wiki
server>* → *Authenticate* (access may require a specific role). Meanwhile, the in-repo source is its
`AGENTS.md`/`CLAUDE.md` and, where it exists, a `docs/configuration.md`.

## 3. Get the value

- Derive it from something you can read (the service's k8s manifest →
  `<svc>.<namespace>.svc.cluster.local`) and say where it comes from.
- Anything you can't derive (a vendor key), ask for verbatim. **Never invent it.**

## 4. Write it to every environment

| Store | How |
|---|---|
| setting | Key under `shared:` if it's the same everywhere, otherwise in **each** environment block. If the app can't boot without it, add it to `REQUIRED_KEYS` (e.g. `config/settings_loader.rb`) |
| sops | `sops credentials/<env>.enc.env` for each environment (or `sops set` for YAML files). Never hand-edit an `ENC[...]` file |
| rails-credentials / gh-secrets | Procedure in `/credentials` |

All in the **same PR** as the code that reads it. A rename is two changes: add the new name and
remove the old one, in every environment. A new secret only reaches the pods after a redeploy.

Don't ask permission to write staging or production, and don't leave "remember to configure X in
prod": do it yourself. **Do confirm** before overwriting or rotating a secret already live in
production, or changing who can decrypt (`sops updatekeys`, KMS/age keys).

## 5. Verify per environment

```bash
sops -d credentials/<env>.enc.env | grep -c NAME   # in every environment; don't print the value
grep -n NAME credentials/<env>.enc.env             # at rest: ENC[AES256_GCM,…]
git diff                                           # no plaintext secret
```

"It's in staging" is not verification. If the value connects to another service, add it to the
**Internal dependencies** table in the app's wiki note (in the wiki repo itself if the MCP is
read-only).
