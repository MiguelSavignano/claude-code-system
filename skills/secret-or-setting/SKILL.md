---
name: secret-or-setting
description: Decide what a new configuration value is — a secret, a non-secret setting, or a runtime switch — before storing it. Has the "is it a secret?" test, examples per category, the default when unsure, how to split a mixed value, and where each kind goes next (secret → /credentials, setting → the app's settings file, switch → /feature-flags). Use when adding or renaming an environment variable or setting, wiring up a new service, or when asked "is this a secret?" or "where does this value go?". Invoke as /secret-or-setting [NAME].
argument-hint: "[NAME]"
---

# secret-or-setting — classify a configuration value

Decide the kind first; the store follows from it. This skill only decides — writing the value is
someone else's job (see [Next](#next)).

## The test

**It's a secret if it grants access.** Posted in a public channel, could someone log in, sign
something, or impersonate the app with it? Yes → secret. No → setting, unless it must change
without a deploy → switch.

| Kind | Examples | Why |
|---|---|---|
| **Secret** | passwords, API keys, tokens, JWT/signing secrets, private keys, `client_secret`, connection strings with a password | grants access |
| **Setting** | service URLs, hostnames, buckets, region, table names, mailboxes, timeouts, limits, `LOG_LEVEL`, `client_id`, a per-environment boolean flag | grants nothing, varies by environment, changes with a deploy |
| **Switch** | kill switches, incident thresholds, per-user or per-company flags | must change at runtime, without a deploy |

## Edge cases

- **Default when unsure whether it grants access: secret.** A secret mistaken for a setting is
  exposed; a setting mistaken for a secret is only hidden from review.
- **But don't encrypt what obviously grants nothing.** A hostname or a table name in a plaintext
  settings file shows up in the PR diff; encrypted, it is hidden without being protected.
- **Mixed value** (`postgres://user:pass@host/db`): split it — host and database are settings, the
  password is a secret. If it can't be split, the whole value is a secret.
- **A fallback default is not a value.** `ENV.fetch('X', 'http://localhost:…')` hides a missing
  setting: staging/production boot "healthy" pointing at localhost. Classify it and store it; don't
  default it at the call site.
- **Every value the code reads must exist in every deployed environment**, in the same change as
  the code that reads it.

## Next

| Kind | Goes to |
|---|---|
| Secret | `/credentials` — which store the app uses and how to write it |
| Setting | the app's plaintext, committed settings file (e.g. `config/settings.yml`) |
| Switch | `/feature-flags` |
