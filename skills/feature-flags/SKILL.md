---
name: feature-flags
description: How a feature flag works in each kind of web app — the advanced system (a DB table of flags in the backend API with rules by email, company, role, IP and environment, created per environment from the admin, read by the frontend with a `useFeatureFlag` hook; a missing row is `false`) and the simple system (one boolean per environment in the repo's non-secret config, usually on in staging and off in production) — and, for each, how a flag is added, turned on in staging and retired. Production is turned on by the ticket owner, never by an agent. Use when adding, turning on or retiring a flag, or when someone asks "how do I hide this in production". Invoke as /feature-flags [app].
---

# feature-flags — how a flag works in each app

Whether a flag is needed: `/gate-policies`.

There are two systems; use the one the repo already has, don't invent a third. Both fail closed:
with no flag defined, the feature is off.

**Turning a flag on in production** is the `delivery-flag` policy in `/gate-policies`: the ticket
owner does it. An agent never turns on a production flag, in either system.

## 1. Advanced system — DB table + admin (backend API + SPA)

- **What it is:** rows in a flags table in the backend API (e.g. `feature_flags` / `FeatureFlag`),
  not code or config. Each environment's database has its own rows.
- **Rules:** a master `enabled` switch plus include/exclude rules by email, company, role, IP range
  and environment. Exclude wins over include; with no include rule it is on for everyone not
  excluded.
- **Missing row = `false`.** So the code can be merged and deployed before any row exists.
- **Managed from the admin** (`/admin/feature_flags` in our reference repo — verify), one flag per
  environment. In our reference repo the API deliberately has no endpoint to create flags (verify):
  don't look for one.
- **Reading:** the frontend uses a hook, `useFeatureFlag(KEY)`, with the key declared in a constant
  of flag keys; the backend asks the model (e.g. `FeatureFlag.enabled?('key', user:, ip:)`). Grep
  for both before saying who consumes a flag.

| Action | How |
|---|---|
| Add | New key in the frontend constant and/or backend check; tests with the flag on **and** off. No row needed to merge |
| Turn on in staging | Create the row in the **staging** admin and enable it (optionally only for some emails or a company) |
| Turn on in production | Ticket owner, from the production admin (`delivery-flag`) |
| Retire | When it is on in production for everyone and the ticket is delivered: the next PR removes the check and the off path, then delete the row in every environment |

## 2. Simple system — a boolean per environment in config (other repos)

A key in the repo's committed, non-secret per-environment config (`config/settings.yml` in Rails,
the app's settings file elsewhere). Usually off by default, on in staging, off in production.
Why it is config and not a secret: `/secret-or-setting`.

```yaml
default:
  # What it hides and why (ticket key).
  REPORT_EXPORT_ENABLED: false
staging:
  REPORT_EXPORT_ENABLED: true
production:
  REPORT_EXPORT_ENABLED: false
```

Write the production value explicitly: a bare `production:` key yields `null`, not an inherited
`false`, in many loaders. Read it with your config loader's boolean cast, and gate the route or
code path on it (e.g. return 404 when off).

| Action | How |
|---|---|
| Add | The key in every environment block + the check; tests with both values |
| Turn on in staging | Already on via the `staging:` block once merged |
| Turn on in production | A PR that flips `production:` to `true`, merged by the ticket owner (`delivery-flag`); there is no live switch |
| Retire | When it is on in production and the ticket is delivered: the next PR deletes the key from every block and the off path |

Don't override the flag with a real environment variable: the deployed value stops showing in the
diff.

## Not a flag

- A flag does not protect a migration or a contract change: `/gate-policies` (`rollout-plan`).
- Whether a value is a secret, a setting or a switch: `/secret-or-setting`. A switch not segmented
  by user (kill switch, threshold) may live in an operational-config table instead of the flags
  table, if the repo has one (verify in the repo).
