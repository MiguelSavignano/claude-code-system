---
name: feature-flags
description: How feature flags work in your web apps — an advanced system (a `feature_flags` table in the backend API with rules by email, company, role, IP and environment, managed from /admin/feature_flags and read by the SPA with `useFeatureFlag`) and the simple flag for the other repos, one boolean per environment in config, usually on in staging and off in production. Use when a feature is merged before it is ready for production, when adding, turning on or retiring a flag, or when someone asks "how do I hide this in production". Invoke as /feature-flags [app].
---

# feature-flags — merge without releasing

**Type:** knowledge + procedure.

When merging to `main` deploys to production (`/gitflow`), a flag separates **deploying** (the
code is in production) from **releasing** (users see it). There are two systems; use the repo's,
don't invent a third. Whether a flag is needed is decided by `/gate-policies`; this skill covers
how.

## 1. Backend API + SPA — advanced flags

Flags are **database rows** in the backend API (e.g. a `feature_flags` table, `FeatureFlag`
model). They are not code or config: each environment (staging, production) has its own.

**Management:** only from the admin panel, `/admin/feature_flags`. The API does not allow
creating or editing them (deliberately).

**Fields:** `key` (unique), `name`, `description`, `enabled` (master switch) and include/exclude
pairs per dimension: `emails`, `company_ids`, `roles`, `ip_ranges` (CIDR) and `environments`.

**Evaluation** (e.g. `app/services/feature_flags/evaluate.rb`), in this order:

1. `enabled = false` → off for everyone.
2. Environment excluded, or there is an `include_environments` and the current one is not in it → off.
3. The user/request matches any `exclude_*` rule → off (deny wins).
4. If there is any `include_*` rule, the user must match at least one; if there is none, on for
   everyone not excluded.

A flag that does not exist evaluates to `false` (fail closed).

**Reading:**

| Where | How |
|---|---|
| Frontend (SPA) | `GET /api/v1/feature_flags/evaluate` → `{flags: {key: bool}}`. Hook `useFeatureFlag(FEATURE_FLAGS.myFlag)` (e.g. `src/hooks/useFeatureFlag.ts`); the key is declared in a `FEATURE_FLAGS` constant (e.g. `src/services/featureFlag.service.ts`) |
| A single key | `GET /api/v1/feature_flags/:key/enabled` |
| Backend (Ruby) | `FeatureFlag.enabled?('my_flag', user: current_user, ip: client_ip)` |

Keep a dated note of which flags exist and who consumes them (frontend, backend, or both) — grep
for the hook and for `FeatureFlag.enabled?` before stating it.

**Do not confuse with:** an `AppConfiguration`-style table (operational switches not segmented by
user — kill switches, thresholds) or `config/settings.yml` (per-environment values that change
with a deploy).

**Flow for a new feature:**

1. Code behind the flag with the new key; tests with the flag on **and** off.
2. Merge → deploys. With no row, the flag is `false`: nothing visible.
3. Create the flag in the **staging** admin, turn it on and test (`/test-apps`).
4. Create the flag in the **production** admin. For a gradual rollout, turn it on first for a few
   emails or one company (`include_emails`, `include_company_ids`), then for everyone.
5. Once it has been stable for everyone for a while: a PR that removes the check and the dead
   code, and deletes the row. Open a cleanup ticket in the tracker (`/task-tracker`) when you create the flag.

Creating or editing a flag in production is a customer-visible change: confirm with the user
before doing it.

## 2. Other repos — simple per-environment flag

A boolean in the repo's **non-secret** config, one per environment. The usual pattern: **off by
default, on in staging, off in production** until validated.

Example (Django with a YAML settings file, e.g. `config/settings.yaml`, section `# --- Feature flags`):

```yaml
default:
  # What it does and why it is off (ticket / doc).
  REPORT_EXPORT_ENABLED: false
staging:
  REPORT_EXPORT_ENABLED: true
production:                       # no key → inherits false from default
```

read in the settings module (e.g. `<project>/settings.py`):

```python
REPORT_EXPORT_ENABLED = config.as_bool('REPORT_EXPORT_ENABLED')
```

```python
if not settings.REPORT_EXPORT_ENABLED:
    raise Http404   # the route "does not exist" in an environment with the flag off
```

- **Turning it on in production = a PR that changes the YAML + deploy.** There is no live switch.
- It is non-secret config: it goes in the committed YAML, **not** in the sops file
  (`/secret-or-setting`, `/credentials`). A real environment variable may take precedence over the
  YAML (check your config loader); don't use one for flags, or the deployed value stops showing in
  the diff.
- Tests with both values (`override_settings(MY_FLAG=True)` / `False`).
- Other repos: same pattern in their per-environment config (`config/settings.yml` in Rails apps).
  If the repo has none yet, follow the example above.
- Retirement same as above: when the flag is on in every environment and stable, delete the key
  and the off code path.

## Which system to use

| You need | Use |
|---|---|
| Hide something in production in the backend API / SPA that have the advanced system | Advanced flag (admin) |
| Enable only for certain users, companies or IPs | Advanced flag (admin) |
| Turn on/off without deploying | Advanced flag, or the operational-config table if it does not depend on the user |
| Hide in production in any other repo | Boolean in per-environment config |

## Rules

- Off by default (fail closed).
- A flag does not replace a rollout plan: destructive migrations or contract changes →
  `/rollout-plan`.
- Every flag is born with its cleanup ticket; a flag that is on everywhere is debt.
