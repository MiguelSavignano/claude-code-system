---
name: credentials
description: The three secret stores an app can use — sops-encrypted file, Rails encrypted credentials, GitHub Actions secrets — how to tell from the repo which one an app uses, and per store the commands to list key names, read one key, add or change a key, rotate, and verify a key exists without printing its value. Also: moving a plaintext `.env` into a store. Production decrypts and writes need the user's confirmation. Use when asked "where are X's secrets", "add this secret", "rotate this key", or before touching `.sops.yaml`, `*.enc*` or `credentials/*.yml.enc`. Invoke as /credentials [app] [env].
argument-hint: "[app] [env]"
---

# credentials — the secret stores

Is it a secret at all? `/secret-or-setting`. Non-secret settings (`config/settings.yml`) and
runtime switches (`/feature-flags`, kill switches included) are not covered here.

## Safety rules

- **Key names are free; values are not.** Listing names needs no confirmation. Decrypting a value,
  or any write, in **production** needs the user's explicit yes in this turn. Never commit unless
  the user asks.
- **Never print a value** to stdout or chat. List names only; check a key with a count
  (`grep -c '^KEY='`), never `grep KEY` on decrypted output.
- **Never type a literal secret in a command** (it lands in shell history): load it into a
  variable from the store and pass the variable. On a shared machine, prefer files read with `@`
  (e.g. `curl -H @headers`) so values don't show in the process list. No scripts in `/tmp`. If a temp file is unavoidable: `mktemp` + `chmod 600`, delete it
  (`trap 'rm -f "$f"' EXIT`).
- **Never hand-edit an `ENC[...]` file** as plaintext: it breaks the MAC.

## 1. Which store does this app use?

Look in the repo; don't infer it from the stack (a Rails app may use sops or gh-secrets).

```bash
ls -a .sops.yaml config/credentials.yml.enc config/credentials/ 2>/dev/null
git ls-files | grep -E '\.enc(\.|$)'                       # encrypted files
grep -rhoE 'secrets\.[A-Za-z0-9_]+' .github/workflows | sort -u  # what CI injects
grep -rln 'RAILS_MASTER_KEY\|SOPS_AGE_KEY' .github/workflows   # how CI decrypts
```

| Evidence | Store |
|---|---|
| `.sops.yaml` + `*.enc.env` / encrypted YAML | **sops** |
| `config/credentials*.yml.enc` **and** the deploy sets `RAILS_MASTER_KEY` | **Rails credentials** |
| App values arrive as `secrets.X` in the deploy workflow, no encrypted file | **GitHub secrets** |

A `credentials.yml.enc` generated with the app but never wired into deploy is not the store. If the
evidence is missing or contradictory, **stop and ask** — don't guess.

## 2. sops

Keys stay plaintext at rest; only values are encrypted. Recipients are in `.sops.yaml`
(`path_regex` must match the file).

```bash
grep -v '^sops_' credentials/<env>.enc.env | cut -d= -f1      # list names, no decryption
sops --decrypt credentials/<env>.enc.env | grep -c '^KEY='    # exists? prints 1 or 0
sops --decrypt --extract '["KEY"]' config/app.yml >/dev/null && echo present   # YAML
sops set credentials/<env>.enc.env '["KEY"]' '"value"'        # add/change (value is JSON)
sops updatekeys credentials/<env>.enc.env                     # after editing .sops.yaml recipients
```

- `sops set` puts the value on the command line: fine for test values; for a real secret ask the
  user to run `sops credentials/<env>.enc.env` (opens `$EDITOR`) in their own terminal.
- Older sops has no `set` subcommand: `sops --set '["KEY"] "value"' FILE`. Check `sops --help`;
  `set` on dotenv files — verify on your version.

## 3. Rails encrypted credentials

`config/credentials/<env>.yml.enc` (per env) or `config/credentials.yml.enc` (shared), opened by
`RAILS_MASTER_KEY` or the `.key` file next to it. Per Rails' security guide, a per-env file takes
precedence over the shared one: adding to the shared file does nothing for that env.

```bash
bin/rails credentials:show --environment <env> | grep -oE '^[a-z_]+:'   # top-level names only
bin/rails credentials:show --environment <env> | grep -c '^key_name:'   # exists?
bin/rails credentials:edit --environment <env>                          # add/change, via $EDITOR
```

- There is no way to list names without decrypting: in production even the commands above need
  confirmation. Never run bare `credentials:show` — it prints every value.
- `credentials:edit` needs a blocking editor: ask the user to run it in their terminal
  (`EDITOR="code --wait" bin/rails credentials:edit --environment <env>`).
- The diff is one opaque blob; verify with the count above, and keep `*.key` in `.gitignore`.

## 4. GitHub Actions secrets

Write-only: values can never be read back.

```bash
gh secret list --repo <org>/<repo> [--env <env>]                    # names only
gh secret list --repo <org>/<repo> --env <env> | grep -c '^KEY\b'   # exists?
gh secret set KEY --repo <org>/<repo> --env <env>                   # prompts for the value
gh secret set KEY --repo <org>/<repo> --env <env> < "$f"            # from a mktemp file
```

Some apps bundle a whole `.env` into one secret (e.g. `PRODUCTION_ENV_VARIABLES`): check the
workflow before assuming one secret per key. A new value reaches the app only on the next deploy.

## 5. Rotate

1. Get the new value from the issuing system (never invent it). 2. Write it to every environment
that uses it (production: confirm first, and say what breaks until consumers update). 3. Redeploy
(`/ci-cd`). 4. Verify with a count. 5. Revoke the old value at the issuer. Rotating who can decrypt
(sops recipients, `RAILS_MASTER_KEY`) affects other people and CI: confirm first.

## 6. Plaintext `.env` in a deployed app

Not a store. Don't add to it; stop and propose the move:

```bash
git ls-files | grep -E '(^|/)\.env'    # committed? then its secrets are exposed
```

Classify each value (`/secret-or-setting`), move secrets into the app's store (section 1; sops if
it has none), gitignore the plaintext file. If it was ever committed, deleting it does not
unpublish it: the secrets need rotating (section 5).

Sources: `sops --help`, `bin/rails credentials:help`, Rails Security Guide (Custom Credentials),
`gh secret --help`.
