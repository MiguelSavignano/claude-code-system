---
name: grafana-dashboards
description: Build or extend a Grafana dashboard (panels + LogQL over the shared Loki datasource) for an app — HTTP traffic panels (status codes, endpoints, outbound-call errors), importer/cronjob panels (run/item events, error rate by pipeline, per-pipeline resource counts, a run_id trace lookup) and outbound webhook panels — using your reference repo's checked-in dashboard (`docs/dashboards/grafana-dashboard.json`) as the model. Dashboards are edited as JSON, checked into the app repo, and applied by committing to `docs/dashboards/grafana-dashboard.json` on the branch that repo's `update-grafana-dashboard.yml` workflow watches — the workflow pushes it to Grafana automatically through the cluster's dashboard-sidecar convention (apps without that workflow need it added, or fall back to the manual Import UI). Use when adding or changing a dashboard panel, or when asked "put this on a dashboard". Invoke as /grafana-dashboards [app] [what to add].
---

# grafana-dashboards — build panels from the `/observability` event contracts

**Type:** procedure (edits dashboard JSON, ships it via a commit).

A dashboard is only as good as the structured logs feeding it — if you're building a
panel for an event that doesn't exist yet or isn't JSON, stop and use **`/observability`**
first (its Step 0 table tells you which apps even emit JSON logs) and a Loki query tool / MCP,
if available, to confirm the event is actually live in Loki before writing a query against it.
A panel with a typo'd field name doesn't error in Grafana, it just silently shows "No data" —
verify the query with a count/query call first (see `/observability`), then paste it into the
panel.

## Reference implementation: your reference repo's dashboard

`<reference-app>/docs/dashboards/grafana-dashboard.json` is the worked example to copy from.
A good reference dashboard has five rows, top to bottom:

1. **HTTP** — status-code rates for specific paths, total request volume by method,
   `http_outbound_error` by host.
2. **Worker activity (legacy logs)** — raw `|~ "regex"` panels over plain log lines, kept
   for workers not yet on the structured contract.
3. **Importer errors (all pipelines)** — `import_item_failed`/`import_run_failed`
   grouped by `pipeline`+`error_class`, generic enough that a **new** pipeline shows up
   automatically with zero dashboard edits.
4. **Importer pipelines — resource/status counts** — one panel per pipeline, hand-built
   because each pipeline logs different resource/count field names.
5. **Webhooks (outbound deliveries)** — outbound webhook fan-out grouped by *subscriber*
   (`owner`) and business event (`webhook_event`), see Step 3b.

Older per-app `docs/dashboards/README.md` files may still say Grafana's Import UI is the
*only* way to apply a dashboard. If your cluster runs the dashboard sidecar described below,
treat any README still saying "Import UI only" as stale; this skill is the source of truth.

## How the live Grafana updates itself (dashboard-sidecar convention)

This applies when Grafana is deployed with the kube-prometheus-stack Helm chart (or any setup
with the Grafana dashboard sidecar enabled). Verify each point on your cluster before relying
on it. The public Grafana URL (`<grafana-host>`) is often SSO-gated with no API access through
it, which can make it look like there is no automation path. There usually is one — it just
doesn't go through that URL:

- Grafana runs as a Deployment/Service in its own namespace (e.g. `prometheus-stack`, service
  `prometheus-stack-grafana`) and can be reached directly with
  `kubectl port-forward -n <grafana-namespace> svc/<grafana-service> 3000:80`. If SSO is
  enforced at the ingress in front of `<grafana-host>` rather than by Grafana itself, this
  bypasses it.
- An internal admin account usually exists independent of SSO: a Secret in that namespace
  (e.g. `prometheus-stack-grafana`) with keys `admin-user`/`admin-password`. Check it works
  with `GET /api/user` through the port-forward (`isGrafanaAdmin: true`).
- The Grafana deployment runs a dashboard sidecar container (`grafana-sc-dashboard` in the
  chart) configured with something like `NAMESPACE=ALL`, `LABEL=grafana_dashboard`,
  `LABEL_VALUE=1`: it watches ConfigMaps matching that label **across every namespace in the
  cluster**, writes their JSON into Grafana's provisioning folder, and calls
  `POST /api/admin/provisioning/dashboards/reload` using the same admin secret internally.
  Check the sidecar's env vars on your deployment — the label name/value may differ. An
  existing labeled ConfigMap in the Grafana namespace is a good sign the pattern is in use.
- This means **no Grafana API token is needed for routine updates** — just a correctly
  labeled ConfigMap, applied by whatever already has `kubectl` access to *any* namespace
  (it doesn't need to be Grafana's own). A successful end-to-end apply looks like: ConfigMap
  applied → sidecar log shows `Writing /tmp/dashboards/<app>-dev.json` →
  `Dashboards config reloaded` → dashboard version incremented, `meta.provisioned: true`.
- A Grafana **service-account token** (HTTP API, `Authorization: Bearer <token>`) is a
  separate, heavier-weight mechanism — useful for *reading* live dashboard state (e.g.
  diffing live vs. checked-in JSON to discover drift) or one-off manual fixes, but don't
  reach for it as the default write path when the ConfigMap route exists and needs no
  long-lived credential in CI. `grafana-cli` (the actual CLI binary) is unrelated to either
  of these — it has no dashboard commands and no remote-auth flag at all; it only manages
  plugins/admin-password-reset on the machine Grafana itself runs on.

## Step 1 — where the dashboard lives and its shared building blocks

- **File**: `<app>/docs/dashboards/grafana-dashboard.json` (the reference repo's path;
  mirror it for a new app so `/grafana-dashboards` and a human both know where to look).
  Add a sibling `README.md` describing each row, like the reference repo's.
- **Datasource**: typically one shared Loki instance for the whole cluster — reuse the `uid`
  seen in the reference repo's panels (`<loki-uid>`) unless you confirm a different one in
  Grafana's **Connections → Data sources**. (A single in-cluster Loki gateway per
  `/observability` suggests one uid; verify per dashboard if unsure.)
- **`$namespace` template variable** — every panel's LogQL uses `namespace="<ns>-$namespace"`.
  `<ns>` is the Loki namespace prefix, **not always the app name**: e.g. a backend API that is
  deployed into its frontend's namespace logs under `web-<env>`, and there is no `api-*`
  namespace in Loki. Check the actual `namespace` label values with your Loki label tool before
  writing a panel; a wrong prefix shows "No data".
  Always `| json | __error__=""`: one non-JSON line aborts a bare `| json`. The variable
  lets one dashboard serve staging and production; define it as a custom variable with values
  `staging,production` (or a Loki label-values query if the app's namespace naming is
  irregular — list the real namespaces with `kubectl get ns` first, see `/kubectl`).
- **`$run_id` template variable** (importer dashboards only) — a free-text/textbox
  variable, empty by default, feeding a trace-lookup **logs** panel
  (`{namespace="<ns>"} | json | __error__="" | run_id=~"$run_id"`) so pasting a `run_id` from an
  error row shows that run's full story.

## Step 2 — HTTP panels

Patterns from the reference dashboard (`namespace`/`service_name` selector per
`/observability`'s namespace note — confirm whether your app shares a namespace with another
service, the way a backend API and its SPA sometimes do):

```logql
# Status-code rate for one endpoint
sum(count_over_time(
  {namespace="<ns>-$namespace", service_name="<app>"} | json | __error__="" | path="/some/path" | status="401" [$__interval]
))

# Total request volume by method
sum by (method) (count_over_time(
  {namespace="<ns>-$namespace", service_name="<app>"} | json | __error__=""
    | method =~ "GET|POST|PUT|PATCH|DELETE|OPTIONS|HEAD" [$__interval]
))

# Exclude noisy/admin paths from an "API traffic" panel
sum(count_over_time(
  {namespace="<ns>-$namespace", service_name="<app>"} | json | __error__=""
    | path !~ "^(/some/noisy/path|/admin.*)$" | method =~ "GET|POST|PUT|PATCH|DELETE|OPTIONS|HEAD" [$__interval]
))

# Outbound HTTP errors by host (see /observability's http_outbound_* contract)
sum by (host) (count_over_time(
  {namespace="<ns>-$namespace", service_name="<app>"} | json | __error__="" | event="http_outbound_error" [$__interval]
))
```

Use panel type `timeseries` for rates over time; a bare stat/`table` panel with the same
query and no `[$__interval]` range vector for a single current number.

## Step 3 — importer / cronjob panels

"Cronjob" here means a scheduled worker (e.g. sidekiq-cron entries in
`config/sidekiq_schedule.yml` in a Rails app), not necessarily a literal k8s CronJob resource;
it logs through the same `import_run_*`/`import_item_*` contract as an on-demand importer.
Check `/observability`'s contract before building these.

**Generic — covers every pipeline without a dashboard edit** (put these once, near the top):

```logql
# Item success vs failure, over time, split by pipeline
sum by (pipeline, event) (count_over_time(
  {namespace="<ns>-$namespace"} | json | __error__="" | event=~"import_item_(finished|failed)" [$__interval]
))

# Errors by pipeline + error_class — the panel that makes a brand-new pipeline show up for free
sum by (pipeline, error_class) (count_over_time(
  {namespace="<ns>-$namespace"} | json | __error__="" | event="import_item_failed" [$__interval]
))
sum by (pipeline, worker, error_class) (count_over_time(
  {namespace="<ns>-$namespace"} | json | __error__="" | event="import_run_failed" [$__interval]
))
```

**Per-pipeline resource/status count** — needs one panel per pipeline because each logs
different field names; copy the pattern, swap `pipeline=` and the field:

```logql
# Outcome breakdown (single-resource or run-level pipelines)
sum by (status) (count_over_time(
  {namespace="<ns>-$namespace"} | json | __error__="" | pipeline="<pipeline>" | event="import_run_finished" [$__interval]
))

# A numeric field from the event, summed (e.g. rows created, emails sent) — unwrap, not count
sum(sum_over_time(
  {namespace="<ns>-$namespace"} | json | __error__="" | pipeline="<pipeline>" | event="import_item_finished"
    | status="processed" | unwrap emails_sent [$__interval]
))
```

`unwrap <field>` turns a numeric JSON field into a summable metric — use it for counts the
event already carries (`records_created`, `fetched`, `duration_ms`) instead of a second
`count_over_time` with a redundant label.

## Step 3b — outbound webhook / fan-out panels

When an app POSTs a business event to registered subscribers, the question asked of the
dashboard is almost always *"how many `<event>` did we send to `<subscriber>`?"* — so the
panels group by the **subscriber identity**, not by URL or host. The contract to expect (see
`/observability`'s webhook reference; in the reference repo it lives in the delivery worker and
is documented alongside its external-services docs): `webhook_delivery_sent` /
`webhook_delivery_failed` / `webhook_subscription_disabled` carry `owner`, `webhook_event`,
`subscription_id`, `target_url` (+ `error_class`, `consecutive_failures` on failures), and
`webhook_published` carries `webhook_event` + `subscriptions` (the fan-out size).

```logql
# The headline: deliveries of specific events to one subscriber, over the picked range
sum by (webhook_event) (count_over_time(
  {namespace="<ns>-$namespace", service_name="<app>-worker"}
    | json | __error__="" | event="webhook_delivery_sent"
    | owner=~"$webhook_owner"
    | webhook_event=~"order.created|invoice.created" [$__range]
))

# Discovery panel — every subscriber, so the real `owner` strings are visible
sum by (owner, webhook_event) (count_over_time(
  {namespace="<ns>-$namespace", service_name="<app>-worker"}
    | json | __error__="" | event="webhook_delivery_sent" [$__interval]
))
```

Three traps worth stating before you build one of these:

- **`owner` is a `$webhook_owner` textbox variable, not `label_values()`** — it is a JSON
  field inside the log line, not a Loki stream label, so Grafana cannot enumerate it. Pair
  the filtered panels with an unfiltered "by owner" panel so the valid values are readable
  off the dashboard.
- **Publisher and deliverer usually run in different pods.** Delivery happens in the job
  runner (e.g. `service_name="<app>-worker"`); the publish event fires wherever the emit site
  is — controllers *and* workers — so its panel needs a selector covering both
  (e.g. `service_name=~"<app>.*"`). One selector for both rows silently drops half the data.
- **`published × subscribers ≠ deliveries` is normal.** Retries log one failure per
  attempt, and a publish with zero subscribers may not be logged at all. Say what a panel
  counts in its `description` rather than letting a reader assume it counts events.

The `http_outbound_*` events (Step 2) also fire for each delivery via the HTTP middleware,
but they only know `host`/`status` — that's the transport view. Keep both: `host` answers
"is the subscriber up", `owner`+`webhook_event` answers "what did we send them".

**Run trace panel** (logs panel, not timeseries):

```logql
{namespace="<ns>-$namespace"} | json | __error__="" | run_id=~"$run_id"
```

## Step 4 — adding a panel for a brand-new pipeline

1. Confirm the worker emits `import_run_*`/`import_item_*` via `/observability` (or fix its
   instrumentation first — a dashboard panel over a plain-text log is a regex, not a `| json` query).
2. **Nothing to do** for the generic error-by-`pipeline` panels in Step 3 — they already
   cover it.
3. Add one dedicated panel under the "resource/status counts" row: copy an existing panel
   JSON of the same shape (e.g. another `sum by (status) (...)` panel), bump `id` to an
   unused integer, set `gridPos` to the next free row (see Step 5), change `pipeline=` and
   the field name(s) in `expr`, update `title`/`description` to name the worker.

## Step 5 — JSON editing mechanics

- **`gridPos`**: 24-column grid. `{h, w, x, y}` — two panels side by side at half-width are
  `w: 12` at `x: 0` and `x: 12`, same `y`. To append a new panel below the last one in a
  row, set its `y` to `(last panel's y + last panel's h)`; getting this wrong just makes
  panels overlap visually, it doesn't error.
- **`id`**: must be unique across the whole dashboard (not just the row) — scan existing
  `"id":` values and pick the next unused integer.
- **`datasource.uid`**: every panel repeats `{"type": "loki", "uid": "<uid>"}` — copy it
  from a sibling panel rather than retyping.
- **`fieldConfig`**: copy the block from an existing panel of the same `type`
  (`timeseries`/`logs`/`stat`) rather than hand-writing Grafana's schema — it's verbose and
  mostly boilerplate (color mode, axis config, thresholds).
- **`targets[].expr`**: the LogQL string. Multi-line `expr` strings in the checked-in JSON
  are fine (Grafana renders them collapsed); keep the same formatting style as neighboring
  panels for a clean diff.
- Validate the file parses before committing: `python3 -c "import json; json.load(open('docs/dashboards/grafana-dashboard.json'))"`.

## Step 6 — apply and commit

Keep a registry of which apps have the auto-apply workflow and which branch it watches.
Fill this in for your organization:

| App | Has `update-grafana-dashboard.yml`? | Watched branch | Notes |
|---|---|---|---|
| `<app>` | yes / no | `main` / `develop` / … | e.g. "default branch moved to `main` but trigger still on `develop` — needs fixing" |

**If the app has the workflow**: commit the edited `docs/dashboards/grafana-dashboard.json` to
the branch that repo's workflow actually watches — **check it, don't assume**:

```bash
gh api repos/<org>/<app>/contents/.github/workflows/update-grafana-dashboard.yml --jq .content \
  | base64 -d | sed -n '/^on:/,/^\(jobs\|env\|concurrency\):/p'
```

A common drift: a repo moves its default branch (e.g. `develop` → `main`) but the workflow
still watches the old one — so a merge to the default branch does **not** apply the dashboard.
In that case use `workflow_dispatch`, and tell the user the trigger needs fixing.
The repo's `.github/workflows/update-grafana-dashboard.yml` triggers on that path, builds
a labeled ConfigMap (ideally via a shared composite action, e.g. in
`<org>/infrastructure/github/actions/update-grafana-dashboard`), and applies it with the same
AWS role/kubectl setup the app's own deploy workflow already uses — no image build, no
`helm upgrade`, no waiting for a deploy. The sidecar picks it up within roughly a minute;
`workflow_dispatch` is also available for a manual re-run.

**For any other app** (workflow not added yet), two options:
1. **Add the workflow** — copy `update-grafana-dashboard.yml` from an app that already has it
   (pick the one whose AWS-auth convention matches the target repo) into the new app's repo,
   swap the `configmap-name`/`dashboard-key`/namespace inputs, and commit. This is the
   preferred fix — it makes the app self-updating for good.
2. **One-off manual apply**, no CI change needed:
   ```bash
   kubectl create configmap <app>-grafana-dashboard \
     --namespace <any-namespace-you-have-access-to> \
     --from-file=<app>-dev.json=docs/dashboards/grafana-dashboard.json \
     --dry-run=client -o json \
   | jq '.metadata.labels.grafana_dashboard = "1"' \
   | kubectl apply --namespace <any-namespace-you-have-access-to> -f -
   ```
   (requires `kubectl` access to the cluster — see `/kubectl` for cloud re-auth)

**Legacy fallback** (Grafana UI, if neither of the above applies): Grafana UI →
**Dashboards → Import** → paste/upload the edited JSON. This **replaces** the live
dashboard — if others may have edited it live since the last export, export the *current*
live version first and diff before overwriting.

Regardless of which path: sanity-check each new/changed panel actually renders data (not
just "valid JSON, wrong field name") — cross-check its `expr` against a Loki count query
per `/observability` if it shows "No data" and you're not sure whether that's a real zero.
And whenever a dashboard gets edited *live* in the Grafana UI rather than through the
checked-in file, export it (**Dashboard settings → JSON Model**) and commit that back — the
checked-in file and the live dashboard must not drift, or the next person's diff (or the next
automated push) is against a stale baseline and silently overwrites the live edit.

## Rules

- Never hand-edit only in the Grafana UI and leave the repo's JSON stale — export and
  commit every time (document this convention in each app's `docs/dashboards/README.md`).
  This matters more when a push to the watched branch auto-applies the checked-in file —
  a stale repo file will silently overwrite a live-only edit on the next push.
- Don't build a `| json` panel over an app/event that isn't confirmed structured JSON in
  Loki yet — check and verify it with `/observability` first.
- Prefer the generic `pipeline`/`error_class` grouping for error panels over one
  hand-built panel per pipeline — only the resource/status **count** panels need to be
  per-pipeline, because their field names genuinely differ.
- The ConfigMap + dashboard-sidecar path (Step 6) is the preferred way to push a dashboard —
  it needs only ordinary `kubectl` access, no Grafana credential at all. Don't reach for a
  Grafana service-account token/API push as the default; that's for reading live state
  (drift-checking against the checked-in file) or a genuine one-off, and it means minting and
  storing a real credential for something that doesn't need one.
- Reuse `gridPos`/`fieldConfig`/`datasource` blocks from sibling panels instead of writing
  Grafana's schema from memory — lower chance of a subtly malformed panel.
- Only apps listed with the workflow in the Step 6 registry auto-apply — don't tell a user for
  another app that "just commit it" will update Grafana until the workflow has actually been
  added there too.
