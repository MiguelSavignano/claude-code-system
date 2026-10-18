---
name: grafana-dashboards
description: Which Grafana panels to build from an app's structured log events, and their LogQL over Loki — HTTP traffic (status codes, endpoints, outbound errors), importer/cronjob (runs, items, error rate per pipeline, run_id trace lookup) and outbound webhook deliveries — plus the template variables they share and where the dashboard JSON lives in the app repo (`docs/dashboards/grafana-dashboard.json`). Use when adding or changing a dashboard panel, or when asked "put this on a dashboard". Invoke as /grafana-dashboards [app] [what to add].
argument-hint: "[app] [what to add]"
---

# grafana-dashboards — panels from log events

**Type:** knowledge. The events, which apps emit JSON logs, the namespace/`service_name`
traps and `| json | __error__=""`: `/observability`. How the JSON reaches Grafana: `/ci-cd`.

A panel can only be as good as the event behind it. A typo in a field name doesn't error; the
panel just shows "No data". Before writing a panel, confirm the event is live in Loki with an
exact count (`/observability`, section 3), then paste the same selector into the panel.

## Where the dashboard lives

- One JSON file per app, checked into the app repo: `docs/dashboards/grafana-dashboard.json`,
  with a sibling `README.md` that describes each row. Copy the shape of your reference repo's file.
- Copy `datasource` (`{"type": "loki", "uid": …}`), `fieldConfig` and `gridPos` from a sibling
  panel instead of writing Grafana's schema from memory. `gridPos` is a 24-column grid; `id`
  must be unique across the whole dashboard.
- Before committing, check that the file parses: `python3 -c "import json; json.load(open('docs/dashboards/grafana-dashboard.json'))"`.

## Template variables

Define all of them in one list (`templating.list`) and only add the ones your panels use:

| Variable | Type | Used by |
|---|---|---|
| `$namespace` (the environment) | custom, `staging,production`, used as `namespace="<ns>-$namespace"` | every panel. `<ns>` is not always the app name: see `/observability` |
| `$run_id` | textbox, empty by default | the importer trace-lookup logs panel |
| `$webhook_owner` | textbox, default `.*` | webhook panels. `owner` is a JSON field, not a stream label, so `label_values()` cannot list it |

## Panel patterns

In the queries below, `S` stands for `{namespace="<ns>-$namespace", service_name=~"<app>.*"} | json | __error__=""`;
expand it before pasting (it is not LogQL).

| Row | Panels | Event (contract in `/observability`) |
|---|---|---|
| HTTP traffic | status-code rate per endpoint, volume by `method`, outbound errors by `host` | request log lines, `http_outbound_error` |
| Importer / cronjob | items finished vs failed by `pipeline`; errors by `pipeline` + `error_class`; one count panel per pipeline; `run_id` trace lookup | `import_run_*`, `import_item_*` |
| Webhooks | deliveries by `webhook_event` for `$webhook_owner`; deliveries by `owner` (to discover valid owners); failures | `webhook_*` (fields: `observability/references/webhook-fanout.md`) |

One query per pattern:

```logql
# HTTP — status codes for one endpoint (timeseries)
sum by (status) (count_over_time(S | path="/some/path" [$__interval]))

# Importer — errors by pipeline and class; a new pipeline shows up with no dashboard edit
sum by (pipeline, error_class) (count_over_time(S | event="import_item_failed" [$__interval]))

# Importer — run trace lookup (logs panel, not timeseries)
{namespace="<ns>-$namespace"} | json | __error__="" | run_id=~"$run_id"

# Webhooks — what we sent to one subscriber over the picked range (stat or bar gauge)
sum by (webhook_event) (count_over_time(S | event="webhook_delivery_sent" | owner=~"$webhook_owner" [$__range]))
```

- **Over time** → `timeseries` with `[$__interval]`. **One number for the picked range** →
  `stat` / `bargauge` with `[$__range]`. A range vector is always required; there is no bare form.
- **Numeric fields** the event already carries (`records_created`, `duration_ms`): use
  `sum_over_time(S | … | unwrap <field> [$__interval])`, not a second `count_over_time`.
- **Per-pipeline count panels** are the only hand-built ones, because each pipeline logs
  different field names. Error panels stay generic (`pipeline`, `error_class`).
- **Webhook selectors:** delivery runs in the worker, publish fires wherever the emit site is
  (web and worker), so `service_name=~"<app>.*"` covers both; a narrower selector drops half.
  `published × subscribers ≠ deliveries` (retries, empty fan-outs): say what each panel counts in
  its `description`.
- Keep the outbound `host` panel next to the webhook panels: `host` answers "is the subscriber
  up", `owner` + `webhook_event` answers "what did we send them".

## How the JSON gets to Grafana

- **The repo has `update-grafana-dashboard.yml`**: committing the file to the branch that
  workflow watches publishes it through the cluster's dashboard sidecar (workflow and shared
  action: `/ci-cd`). Read the workflow's `on:` before saying "merge it and it's live"; verify in
  your cluster that the sidecar picked it up.
- **No workflow**: Grafana → **Dashboards → Import**, and suggest adding the workflow (`/ci-cd`).
  Import replaces the live dashboard: export the live JSON first and diff.
- An edit made live in the Grafana UI must be exported (**Dashboard settings → JSON Model**) and
  committed, or the next publish overwrites it.

## Before calling the panel done

- [ ] The event exists in Loki (exact count) with the same selector the panel uses.
- [ ] Every `$variable` in an `expr` is defined in `templating.list`.
- [ ] The panel shows data in Grafana for staging (or a real zero you checked with a count).
- [ ] The JSON parses and is committed; no live-only edits left behind.
