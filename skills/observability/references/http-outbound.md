# Contract A — outbound HTTP calls (`http_outbound_*`)

One event before sending, one after receiving (or on error). Implement it once per HTTP client
library as a middleware/hook, not per call site.

| event | extra keys | severity |
|---|---|---|
| `http_outbound_request` | `method`, `host`, `path`, `payload_bytes` (omitted if no body) | info |
| `http_outbound_response` | `method`, `host`, `path`, `status`, `duration_ms` | info |
| `http_outbound_error` | `method`, `host`, `path`, `duration_ms`, `error_class`, `error_message` (see below) | error |

What never goes in these events (rules: `/observability`, section 2):

- **No full URL.** `path` is the path **without the query string**; tokens and signatures live there.
- **No body.** Size (`payload_bytes`) or shape, never the content.
- **No credentials.** Header names only, never values (`Authorization`, `Cookie`, API keys).
- **`error_message`** only if the client's error cannot echo the URL or the body; otherwise omit it.

## Per-stack wiring

- **Ruby (Faraday)** — one middleware registered on every client.
- **Node (axios)** — request/response interceptors.
- **Python (httpx)** — event hooks on the client.

Keep the event names **identical** across languages: that is what lets one panel group by `host`
across apps.

## Verifying it lands

An exact count, with the app's real namespace and `service_name` (`/observability`, section 3):

```logql
sum by (host) (count_over_time({namespace="<ns>-staging", service_name=~"<app>.*"} | json | __error__="" | event="http_outbound_error" [24h]))
```
