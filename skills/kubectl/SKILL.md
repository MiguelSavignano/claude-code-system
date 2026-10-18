---
name: kubectl
description: Common kubectl operations on an app's namespace — pick the right cluster with `kubectl config get-contexts` / `use-context` first, then list pods, read logs, describe failures, exec into a pod, run a one-off console command, and check migrations (including whether the app uses the common `<app>-migration(s)` Job pattern). Use when asked to "check the pods", "read the logs in staging", "open a console" or "why did the migration fail". Invoke as /kubectl [app] [staging|production] [what].
argument-hint: "[app] [staging|production] [what]"
---

# kubectl — common operations

## 1. Context first, always

kubectl acts on whatever context is current, and that can be another cluster or another
environment. Before the first command, look and choose:

```bash
kubectl config get-contexts                 # '*' marks the current one
kubectl config current-context
kubectl config use-context <context>        # the cluster you actually want
kubectl get ns                              # find the app's namespace (often <app>-<env>)
```

Then pass `-n <namespace>` on every command (examples below use `-n $NS`).

- If a call fails with an expired-session / credentials error, ask the user to log in
  (`! aws sso login`, `! gcloud auth login`, `! az login`) and wait — don't retry blindly.
- A **production** namespace or context: ask the user to confirm before any command, even a read.

## 2. Pods and deployments

```bash
kubectl get pods -n $NS
kubectl get deploy,job,cronjob -n $NS
kubectl describe pod <pod> -n $NS              # events: image pull, OOMKilled, failing probes
kubectl get events -n $NS --sort-by=.lastTimestamp | tail -20
kubectl rollout status deploy/<app> -n $NS
```

## 3. Logs

```bash
kubectl logs deploy/<app> -n $NS --tail=100
kubectl logs <pod> -n $NS --tail=100 -f          # follow
kubectl logs <pod> -n $NS --previous             # the crashed container's last run
kubectl logs <pod> -n $NS -c <container>         # pods with sidecars (worker, cron…)
kubectl get pod <pod> -n $NS -o jsonpath='{.spec.containers[*].name}'   # list containers
```

## 4. Exec and one-off commands

Pick an app pod, not a worker, cronjob or migration pod:

```bash
POD=$(kubectl get pods -n $NS -o name | grep <app> | grep -vE 'migrat|cron|sidekiq|worker' | head -1)
kubectl exec -it "$POD" -n $NS -- sh
```

### Rails apps

```bash
kubectl exec -it "$POD" -n $NS -- bundle exec rails c                          # console
kubectl exec "$POD" -n $NS -- bundle exec rails runner 'puts User.count'       # one-off
kubectl exec "$POD" -n $NS -- bundle exec rails db:migrate:status | tail -15   # migrations
kubectl exec "$POD" -n $NS -- bundle exec rake -T | head -30                   # available tasks
```

### Django apps

```bash
kubectl exec -it "$POD" -n $NS -- python manage.py shell                       # console
kubectl exec "$POD" -n $NS -- python manage.py shell -c 'from django.contrib.auth import get_user_model; print(get_user_model().objects.count())'
kubectl exec "$POD" -n $NS -- python manage.py showmigrations | grep '\[ \]'   # pending migrations
kubectl exec "$POD" -n $NS -- python manage.py check --deploy                  # settings sanity
kubectl exec "$POD" -n $NS -- python manage.py help                            # available commands
```

## 5. Migrations

A common pattern is a **Job named after the app** — `<app>-migration` or `<app>-migrations` —
often a Helm pre-install/pre-upgrade hook. **Don't assume it: check whether this app uses it.**

```bash
kubectl get jobs -n $NS | grep -i migrat
```

- **Job exists** → its logs show the real error of a failed migration:
  ```bash
  kubectl logs job/<app>-migrations -n $NS --tail=80
  kubectl describe job/<app>-migrations -n $NS
  ```
  If the Job is gone, its hook delete policy probably removed it; the deploy logs in CI are then the
  source.
- **No Job** → migrations run elsewhere (an init container, the app's entrypoint, a CI step).
  Check `kubectl describe pod` for init containers, or ask the user. Either way, the migration
  status from an app pod is in "Rails apps" / "Django apps" above.

## Rules

- Prefer read-only commands (get, describe, logs, migration status). Anything that changes the
  cluster or the database (delete, rollout restart, scale, `db:migrate`) needs the user's go-ahead.
- Shipping is `/ci-cd`; logs and traces across pods are `/observability`.
