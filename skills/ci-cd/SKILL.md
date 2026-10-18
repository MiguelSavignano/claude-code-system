---
name: ci-cd
description: Which GitHub Actions pipelines your web apps have — unit tests and lint on the PR, a check-only Docker build, which workflow deploys each app and from which branch, the chain on merge to main (staging → e2e → production), how to trigger a deploy by hand with gh, how secrets are decrypted with sops, and the shared composite actions in your infrastructure repo. Use when someone asks "what runs on the PR", "why didn't this deploy", "deploy this branch to staging", or before touching or creating a workflow. Invoke as /ci-cd [app].
---

# ci-cd — the workflows

**Type:** knowledge. Reference: **your reference repo's `.github/`** (`<reference-app>/.github/`);
for a new repo, copy that shape instead of inventing another. The branch, PR and merge process:
`/gitflow`.

## On the pull request

| Workflow | What it does |
|---|---|
| `ci.yml` | Unit tests + lint. Only if the PR is not a draft and touches stack code (`paths:`). Cancels the previous run of the same PR |
| `docker-build.yml` | Builds the image **without push**, so a broken `Dockerfile` or dependency fails on the PR and not on deploy |

Unit tests per repo — fill this in for your organization. Example rows:

| App | Command in CI |
|---|---|
| api (Rails) | `bundle exec rubocop` (job `lint`) + RSpec (job `test`, with a Postgres service, e.g. PostGIS). Slow specs (e.g. PDF rendering) can live in a separate manual workflow (`ci-pdf.yml`) |
| web (SPA) | `npm run lint`, `npm run build` and Playwright against the local dev server talking to the staging API |
| <app> (Python) | `uv run pytest` (with Postgres) |
| worker (Python) | `pipenv run pytest` (e.g. `ci-pr.yaml`, PRs to `develop`/`master`) |

## What deploys each app

Fill this in for your organization. Example rows show the common variants:

| App | Staging | Production | Triggered by | Gate before production |
|---|---|---|---|---|
| api | `staging-deployment.yml` | `production-deployment.yml` | push to `main` → `deploy.yml` | API e2e (specs may live in another repo) |
| web | `staging-deployment.yml` | `production-deployment.yml` | push to `main` → `deploy.yml` | Playwright e2e |
| <app> | `staging-deployment.yaml` | `production-deployment.yaml` | push to `main` → `deploy.yaml` (staging also on push to `develop`) | staging smoke test |
| worker | `deploy-staging.yaml` | `deploy-production.yaml` | push to `develop` → staging; push to `master` → production | none |

Record the date you last checked this against the workflows. When in doubt, read the workflow's
`on:` before stating it.

## On merge to `main`: `deploy.yml`

A single workflow orders the release, all on the same SHA (`github.sha`) and queued
(`concurrency: deploy`, never cancelled):

```
staging-deployment.yml ──► e2e.yml (against staging) ──► production-deployment.yml ──► report
```

- The deploy workflows build and push the image to the registry (e.g. ECR), decrypt the secrets
  and run `helm upgrade --install --atomic` on the cluster (`<cluster>`).
- If e2e fails, production is not deployed.
- AWS authenticates via OIDC (`permissions: id-token: write` in both the calling and the called
  workflow).

## Manual deploy

All deploy workflows accept `workflow_dispatch` with any branch:

```bash
gh workflow run staging-deployment.yml --repo <org>/<app> --ref <branch>
gh run list --repo <org>/<app> --workflow staging-deployment.yml -L 1   # run id
gh run watch <run-id> --repo <org>/<app> --exit-status
```

- **Staging:** free to use, it is for testing a branch before merging. Warn that the next merge to
  `main` overwrites it.
- **Production by hand:** only during an incident or with an explicit reason from the user, and
  confirming it every time. Normally it arrives through the merge. A branch not in `main` deployed
  to production is lost with the next merge.
- Report the result with the run link; if it fails, the job should leave the pod status in the
  summary (e.g. a `fetch-crash-logs` step) and `/kubectl` helps look further.

## Secrets: sops

Each app keeps its secrets encrypted in the repo (e.g. `credentials/<environment>.enc.env`). The
deploy job decrypts them with `sops -d` and applies them as a Kubernetes Secret that the pods
receive as environment variables; that is why a new secret only reaches the pods with a redeploy.
Reading or editing secrets: `/credentials`; adding a variable across all environments:
`/secret-or-setting`.

## Other workflows

| Workflow | What it does |
|---|---|
| `update-grafana-dashboard.yml` | Publishes `docs/dashboards/grafana-dashboard.json` to Grafana → `/grafana-dashboards` |
| `update-grafana-alerts.yml` | Publishes the Grafana alert rules |

## Shared actions

Keep the composite actions every workflow uses (e.g. `setup-sops`, `setup-deployment-tools`,
`helm-deploy`, `update-grafana-dashboard`, `update-grafana-alerts`) in one place, e.g.
**`github.com/<org>/infrastructure` → `github/actions/`**, with the list and usage in that repo's
README. Reference them by ref (`uses: <org>/infrastructure/github/actions/<action>@main`): do not
copy one into the app repo.
