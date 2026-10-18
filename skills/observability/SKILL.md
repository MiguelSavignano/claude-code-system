---
name: observability
description: How an app's logs and traces must look so they can be found, and how to read them — one JSON line per event, the `http_outbound_*`, `import_run_*`/`import_item_*` and `webhook_*` log contracts, what never goes in a log (secrets, personal data, query strings, payload bodies), linking logs to traces by the active span's `trace_id`, exporting OpenTelemetry traces to the cluster collector, and reading logs in Loki (namespace/service_name selectors, `| json | __error__=""`) and traces in Tempo with TraceQL. Use when adding or changing what an app logs, instrumenting a job or importer, adding tracing, or investigating an error in logs or traces. Invoke as /observability [app] [what].
argument-hint: "[app] [what]"
---

# observability — logs (Loki) and traces (Tempo)

Panels built on these events: `/grafana-dashboards`. Logs of one pod with kubectl: `/kubectl`.

## 1. Logs: one event = one JSON line

Free text (`logger.info "Starting import"`) cannot be filtered or aggregated with `| json`.

- One JSON line per event, stable `snake_case` keys and an `event` field (`<domain>_<past_tense_verb>`).
- Anything you will filter or aggregate on is a top-level key, not part of a message string.
- Keys are an API: adding is free; renaming silently breaks panels and alerts.
- Check the **process** emits JSON, not just the app: a worker often configures logging apart
  from the web process. Read how the process you are instrumenting sets up its logger.

Reuse a contract before inventing one:

| If you log… | Contract | Fields |
|---|---|---|
| a call to an external service | `http_outbound_request` / `_response` / `_error` | `references/http-outbound.md` |
| a job, importer or cron that loops over rows | `import_run_*` + `import_item_*` (`pipeline`, `run_id`) | `references/importer-events.md` |
| a fan-out to named subscribers | `webhook_*` (by `owner`) | `references/webhook-fanout.md` |

## 2. What never goes in a log

Logs are read by more people than the database, and kept outside it.

- **No secrets**: tokens, passwords, API keys, `Authorization`/`Cookie` values (header *names* are fine).
- **No personal data**: names, emails, phone numbers, addresses, free-text user input. Log an id.
- **Strip the query string** from every logged URL (`url`, `target_url`): it often carries a token
  or a signature. Log `host` + `path`.
- **No request/response bodies** (`payload`): log their size, shape (keys, item count) and ids.
- **`error_message`** only after checking it cannot echo user data (validation and DB errors
  often do); otherwise log `error_class` alone.

## 3. Reading logs — Loki

Use a Loki query tool / MCP if one is available (count, query, labels…), Grafana Explore otherwise.

- **The namespace is not always the app's own**: an API may log in a sibling app's namespace. A
  made-up namespace returns 0 lines, which looks like "no logs". When in doubt, list the values of
  the `namespace` label.
- **One app is several `service_name`s** (web, worker, cron): select with `service_name=~"<app>.*"`.
- **Always `| json | __error__=""`**: lines that are not JSON get a parse-error label instead of
  fields; the filter drops them so they don't pollute or break the query.
- **"How many?" → an exact count**. Sampled query tools have a cap: their number is a minimum.

```logql
sum by (pipeline) (count_over_time({namespace="<ns>-staging", service_name=~"<app>.*"} | json | __error__="" | event="import_item_failed" [24h]))
```

## 4. Traces: OpenTelemetry → Tempo

1. **`trace_id` and `span_id` on every log line, taken from the active span.** Never generate
   them, and never reuse another id (a `run_id`, a request UUID) as `trace_id`: a W3C trace id is
   32 hex characters, and Grafana's log-to-trace link only works with the real one.
2. Export OTLP to the cluster collector via `OTEL_EXPORTER_OTLP_ENDPOINT`; without it the app
   does not trace, which keeps dev and test untouched. Configure through `OTEL_*` env vars, not code.
3. `OTEL_SERVICE_NAME` must match the logs' `service_name`, or the trace-to-logs jump comes up empty.
4. Do not trace health/liveness/readiness endpoints.

In our cluster (example — verify in the infrastructure repo before relying on it): the collector
runs per node (DaemonSet), so the endpoint is the node IP (`status.hostIP`), not a Service DNS
name; the chart exposes `otel.enabled` / `otel.samplerRatio`; Tempo retention is short, so check
the time range first when TraceQL returns nothing.

## 5. Reading traces — TraceQL

Grafana → **Explore → Tempo → TraceQL** (or a Tempo tool/MCP if available).

```traceql
{resource.service.name="<app>" && span.http.response.status_code>=500}
{resource.service.name="<app>" && span.url.path=~"/api/v1/.*" && duration>1s}
```

From a Loki line, `trace_id` opens the trace; from a span, "Logs for this span" goes back to Loki.
A run's spans can be found by `run_id` only if the app sets it as a span attribute — check the
attribute name in the code before querying it.

## Before calling the instrumentation done

- [ ] One JSON line, stable `event`, filterable top-level fields.
- [ ] No secrets, personal data, query strings or bodies (section 2).
- [ ] The specific process (web/worker) emits JSON; `trace_id` comes from the active span.
- [ ] Deployed and **seen in Loki** with an exact count in staging; the trace shows up in Tempo.
