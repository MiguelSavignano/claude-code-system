# Skills

Custom Claude Code skills that enforce backend architecture and HTTP tracing standards.

---

## `backend-architecture`

Enforces a 4-layer architecture across Node.js, Python, and Ruby on Rails.

```
HTTP Request
     │
     ▼
┌─────────────┐   1. called first — allow or throw
│    Guard    │   ✗ must not return business data
└──────┬──────┘
       │ (passes if authorized)
       ▼
┌─────────────┐   2. parse req, call service, return res
│  Controller │   ✗ no business logic / DB / HTTP calls
└──────┬──────┘
       │
       ▼
┌─────────────┐   3. business logic only
│   Service   │   ✗ no req/res objects
└──────┬──────┘
       │
       ▼
┌─────────────┐   4. external API calls
│     Lib     │   ✗ no business logic
└─────────────┘   ✓ traced HTTP required
```

> Guard is invoked **before any logic**: it is the first call inside the controller, not a downstream step.

| Layer | File pattern | Responsibility |
|-------|-------------|---------------|
| Controller | `*_controller.ts/py/rb` | HTTP in → HTTP out |
| Service | `*_service.ts/py/rb` | Business logic |
| Lib | `lib/**`, `src/lib/**` | External HTTP / SDKs |
| Guard | `guards/**` | Auth / pre-conditions |

**Triggers on:** controller/service/lib/guard files, `app.ts`, `main.py`, `Gemfile`, `package.json`, Rails/Express/FastAPI/Flask/NestJS commands.

---

## `http-outbound-tracing`

Every outbound HTTP call from a lib file must emit structured logs in this format:

```
[HTTP_OUTBOUND] > POST https://api.stripe.com/v1/charges
[HTTP_OUTBOUND] < 200 https://api.stripe.com/v1/charges (42ms) {"id":"ch_123"}
[HTTP_OUTBOUND] ERROR https://api.stripe.com/v1/charges connection refused
```

| Runtime | Implementation | Entry point |
|---------|---------------|-------------|
| Node.js | axios interceptors | `lib/http_client.ts` → `createTracedClient()` |
| Python | httpx event hooks | `lib/http_client.py` → `create_traced_client()` |
| Rails (A) | Faraday middleware | `lib/http_outbound_logger.rb` + `lib/http_client.rb` |
| Rails (B) | httplog gem | `config/initializers/httplog.rb` (global, zero-code) |

> Rails Option A = fine-grained per-client control. Option B = fastest global setup with zero code changes.

**Triggers on:** `lib/http_client.*`, `*_client.ts/js/py/rb`, `lib/**`, `src/lib/**`, axios/httpx/httplog install commands.
