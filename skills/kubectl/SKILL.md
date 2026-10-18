---
name: kubectl
description: Inspect and operate an app's namespace on a Kubernetes cluster with kubectl — read logs, list pods, exec into app pods, run one-off console commands, check migration status. ALWAYS asks the user to (re)authenticate to the cloud provider before any kubectl call. Staging is the default; production requires explicit confirmation every time. Use when asked to "check the pods", "read the logs in staging", "open a console", "why did the migration fail". Invoke as /kubectl [app] [staging|production] [what to inspect or run].
argument-hint: "[app] [staging|production] [what]"
---

# kubectl — operate an app's namespace

**Type:** procedure (runs commands against the live cluster).

Operate any app's namespace on the cluster `<cluster>`: read logs, list pods, exec into app pods,
run one-off commands, check migration status, etc.

## App → namespace map

Namespaces, container names and sidecars belong in each app's note in **your team wiki** (e.g.
`Architecture/apps/<app>.md`) — **resolve them from there**, don't hardcode them in this skill.
A common convention is `<app>-<env>`, set per repo through CI variables such as
`STAGING_K8S_NAMESPACE` / `PRODUCTION_K8S_NAMESPACE`.

With `APP` = the argument or the current repo's name:

1. Read the app note (through a wiki MCP if available, otherwise the wiki repo or site).
2. If it doesn't exist under that name — an alias like `api` or `dashboard` — search the wiki for
   the app name first.
3. From the note, take the namespace (often visible in its monitoring/Grafana links and any prose
   exceptions) and the app + sidecar container names.

> Namespaces usually follow `<app>-<env>`, but watch for exceptions: an app whose image or namespace
> kept an old name after a rename, or a checked-in workflow that targets a stale namespace. Always
> confirm by listing namespaces once authenticated, and if reality differs, **update the app note**:
> ```bash
> kubectl get ns | grep -E 'staging|production'
> ```

## Critical rules (do not skip)

1. **Always (re)authenticate *before* the first kubectl command.**
   Cloud SSO sessions expire often; an expired session makes every kubectl call fail (on AWS EKS:
   `Your session has expired. Please reauthenticate` and `exec: executable aws failed with exit code 255`).
   So **before running any kubectl command**, ask the user to log in *in this session* and wait:

   > Please run `! aws sso login` (or your provider's equivalent, e.g. `! gcloud auth login`,
   > `! az login`) so I can reach the cluster, then tell me when it's done.

   Do not run kubectl until they confirm. If a later call returns an expired-session error, stop and
   ask them to log in again — never retry blindly.

2. **Staging is the default target.** Operate against `<app>-staging` unless production is requested.

3. **Production requires explicit confirmation — every time.**
   NEVER run a command against a `*-production` namespace, and NEVER `set-context` to it, without
   first **stopping and asking the user to confirm in this turn** — even for read-only commands.
   Only after an explicit yes:
   - scope each command with `-n <app>-production` explicitly (do not `set-context` to it),
   - state clearly in your reply that you are running against **production**, and
   - keep it to exactly what was approved.

## Setup (once per session, after login)

Make sure the kube-context is the right cluster and pin the namespace so later commands don't need `-n`:

```bash
kubectl config get-contexts                          # find the context for <cluster>
kubectl config use-context <cluster>                 # if not already current
kubectl config set-context --current --namespace=<app>-staging
```

(Or pass `-n <app>-staging` on each command instead of pinning.)

## Common operations

```bash
# Pods / deployments
kubectl get pods
kubectl get deploy
kubectl describe pod <pod>          # events: image pull errors, OOMKilled, failing probes

# Logs
kubectl logs <pod> --tail=100
kubectl logs deploy/<app> --tail=100
kubectl logs <pod> --previous       # the crashed container's last run
```

### Rails apps

```bash
# Migration hook (e.g. a Helm pre-upgrade Job) — if retained on failure, it shows the real error
kubectl logs job/<app>-migrations --tail=80

# Exec / one-off rails (pick an app pod, not a worker/cronjob/migration pod)
POD=$(kubectl get pods -o name | grep <app> | grep -vE 'migrat|cronjob|sidekiq|worker' | head -1)
kubectl exec "$POD" -- bundle exec rails db:migrate:status | tail -15
kubectl exec -it "$POD" -- bundle exec rails c          # interactive console
```

Other stacks follow the same shape: e.g. `python manage.py showmigrations` / `python manage.py shell`
for Django, or the app's own CLI.

If a pod runs sidecar containers (cron, scheduler, worker), target one with `-c <container>`; take
the exact container names from the app note.

## Guidance

- Prefer **read-only** commands (get/describe/logs/migration status/SELECTs). Treat anything that
  mutates cluster or DB state as an action needing the user's intent — confirm before running it.
- If your migration hook uses a `hook-delete-policy` that keeps the pod on failure, after a failed
  deploy `kubectl logs job/<app>-migrations` shows the real migration error.
- Pair with `/ci-cd` (to ship) and `/observability` (logs and traces across pods); verify the live
  URL afterwards from the app note.
