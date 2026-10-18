---
name: http-outbound-tracing
description: HTTP outbound tracing setup for Node.js (axios), Python (httpx), and Ruby on Rails (Faraday/httplog). Enforces [HTTP_OUTBOUND] log format on all external HTTP calls.
metadata:
  priority: 10
  pathPatterns:
    - 'lib/http_client.*'
    - 'lib/http_logger.*'
    - 'src/lib/http_client.*'
    - 'config/initializers/httplog.rb'
    - 'config/initializers/http_logger.rb'
    - '*_client.rb'
    - '*_client.ts'
    - '*_client.js'
    - '*_client.py'
    - 'lib/**'
    - 'src/lib/**'
    - 'app/lib/**'
  bashPatterns:
    - 'gem\s+["\x27]httplog["\x27]'
    - 'pip\s+install\s+httpx'
    - 'npm\s+install.*axios'
    - 'yarn\s+add.*axios'
    - 'pnpm\s+add.*axios'
---

# HTTP Outbound Tracing

Every outbound HTTP call from any lib file MUST be traced using this exact log format:

```
[HTTP_OUTBOUND] > METHOD https://api.example.com/path
[HTTP_OUTBOUND] < 200 https://api.example.com/path (42ms) {"id":"ch_123"}
[HTTP_OUTBOUND] ERROR https://api.example.com/path connection refused
```

Log only the first 500 chars of the response body. Never log secrets or auth headers.

---

## Node.js — axios interceptors

Create once at `lib/http_client.ts` (or `src/lib/http_client.ts`). All lib files use this instead of raw `axios`.

```ts
// lib/http_client.ts
import axios, { AxiosInstance } from 'axios';

export function createTracedClient(baseURL?: string): AxiosInstance {
  const client = axios.create({ baseURL });

  client.interceptors.request.use((config) => {
    (config as any).__startTime = Date.now();
    console.log(`[HTTP_OUTBOUND] > ${config.method?.toUpperCase()} ${config.baseURL ?? ''}${config.url}`);
    return config;
  });

  client.interceptors.response.use(
    (response) => {
      const ms = Date.now() - (response.config as any).__startTime;
      const body = JSON.stringify(response.data).slice(0, 500);
      console.log(`[HTTP_OUTBOUND] < ${response.status} ${response.config.url} (${ms}ms) ${body}`);
      return response;
    },
    (error) => {
      const url = error.config?.url ?? 'unknown';
      const msg = error.response
        ? `${error.response.status} ${JSON.stringify(error.response.data).slice(0, 200)}`
        : error.message;
      console.error(`[HTTP_OUTBOUND] ERROR ${url} ${msg}`);
      throw error;
    }
  );

  return client;
}
```

**Usage in any lib file:**

```ts
// lib/stripe_client.ts
import { createTracedClient } from './http_client';

const client = createTracedClient('https://api.stripe.com');

export async function createStripeChargeLib(amount: number, token: string) {
  const res = await client.post('/v1/charges', { amount, source: token });
  return res.data;
}
```

---

## Python — httpx event hooks

Create once at `lib/http_client.py` (or `app/lib/http_client.py`). All lib files use this instead of raw `httpx.AsyncClient()`.

```python
# lib/http_client.py
import httpx
import time
import json
import logging

logger = logging.getLogger(__name__)


def _log_request(request: httpx.Request) -> None:
    request.extensions["start_time"] = time.monotonic()
    logger.info(f"[HTTP_OUTBOUND] > {request.method} {request.url}")


def _log_response(response: httpx.Response) -> None:
    start = response.request.extensions.get("start_time", time.monotonic())
    elapsed_ms = int((time.monotonic() - start) * 1000)
    try:
        body = json.dumps(response.json())[:500]
    except Exception:
        body = response.text[:500]
    logger.info(f"[HTTP_OUTBOUND] < {response.status_code} {response.url} ({elapsed_ms}ms) {body}")


def create_traced_client(**kwargs) -> httpx.AsyncClient:
    return httpx.AsyncClient(
        event_hooks={"request": [_log_request], "response": [_log_response]},
        **kwargs,
    )
```

**Usage in any lib file:**

```python
# lib/stripe_client.py
from lib.http_client import create_traced_client

async def create_stripe_charge_lib(amount: int, token: str):
    async with create_traced_client(base_url="https://api.stripe.com") as client:
        try:
            r = await client.post("/v1/charges", data={"amount": amount, "source": token})
            r.raise_for_status()
            return r.json()
        except httpx.HTTPError as e:
            logger.error(f"[HTTP_OUTBOUND] ERROR {e.request.url} {e}")
            raise
```

---

## Ruby on Rails — Option A: Faraday middleware

Create once at `lib/http_outbound_logger.rb`. Register as a Faraday middleware and use via a shared builder.

```ruby
# lib/http_outbound_logger.rb
require 'faraday'
require 'json'

class HttpOutboundLogger < Faraday::Middleware
  def on_request(env)
    env[:start_time] = Time.now
    Rails.logger.info "[HTTP_OUTBOUND] > #{env.method.upcase} #{env.url}"
  end

  def on_complete(env)
    ms = ((Time.now - env[:start_time]) * 1000).round
    body = begin
      JSON.parse(env.response_body.to_s).to_json[0, 500]
    rescue JSON::ParserError
      env.response_body.to_s[0, 500]
    end
    Rails.logger.info "[HTTP_OUTBOUND] < #{env.status} #{env.url} (#{ms}ms) #{body}"
  rescue => e
    Rails.logger.error "[HTTP_OUTBOUND] ERROR #{env.url} #{e.message}"
    raise
  end
end

Faraday::Response.register_middleware http_outbound_logger: HttpOutboundLogger
```

**Shared builder at `lib/http_client.rb`:**

```ruby
# lib/http_client.rb
require_relative 'http_outbound_logger'

class HttpClient
  def self.build(base_url)
    Faraday.new(url: base_url) do |f|
      f.request :url_encoded
      f.response :json
      f.use :http_outbound_logger
    end
  end
end
```

**Usage in any lib file:**

```ruby
# lib/stripe_client.rb
class StripeClient
  def self.create_charge(amount:, token:)
    conn = HttpClient.build('https://api.stripe.com')
    conn.post('/v1/charges', amount: amount, source: token)
  end
end
```

---

## Ruby on Rails — Option B: httplog gem (simpler)

Add to Gemfile and configure in an initializer. Works automatically for all Net::HTTP, Faraday, HTTParty, and RestClient calls.

```ruby
# Gemfile
gem 'httplog'
```

```ruby
# config/initializers/httplog.rb
HttpLog.configure do |config|
  config.logger        = Rails.logger
  config.log_request   = true
  config.log_response  = true
  config.log_headers   = false  # never log auth headers
  config.log_data      = true
  config.prefix        = '[HTTP_OUTBOUND]'
  config.color         = false
end
```

Choose **Option A** (Faraday middleware) when you need fine-grained control per client or when truncating response bodies. Choose **Option B** (httplog gem) for the fastest setup with global coverage.
