# Contract A — outbound HTTP calls (`http_outbound_*`)

One event before sending, one after receiving (or on error). Reference implementation: e.g. a
Faraday middleware in your reference repo (`app/lib/faraday/http_outbound_logger.rb`).

| event | extra keys | severity |
|---|---|---|
| `http_outbound_request` | `method`, `host`, `path`, `url`, `payload` (truncated ≤500 chars, omitted if nil) | info |
| `http_outbound_response` | `method`, `host`, `path`, `url`, `status`, `duration_ms` | info |
| `http_outbound_error` | `method`, `host`, `path`, `url`, `duration_ms`, `error_class`, `error_message` | error |

**Never log credentials** — no tokens, no passwords, no `Authorization` values. Header
*names* only, never values.

## Per-stack wiring

- **Ruby** — register one `:http_outbound_logger` Faraday middleware and reuse it. Don't hand-roll it per client.
- **Node (axios)** — request/response interceptors emitting the same three event names.
- **Python (httpx)** — event hooks on the client, same names.

The shape is language-agnostic; only the transport hook differs. Keep the event names
**identical** across languages — that is what lets one Grafana panel group by `host`
across apps.

## Verifying it lands

With an exact-count Loki query (a Loki tool / MCP if available), using the app's real namespace
(see `/observability`):

```
count  logql: '{namespace="<app>-staging", service_name=~"<app>.*"} | json | __error__="" | event="http_outbound_error"'
       by: ["host"]   since: 24h
```
