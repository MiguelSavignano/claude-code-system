---
name: test-apps
description: Smoke-test one of your apps' web UI or JSON API against staging (or local). Routes to the right tool per surface — Playwright for browser/UI surfaces (an SPA dashboard, a server-rendered admin panel, an SSO-protected web app), curl for JSON APIs (JWT- or API-key-authenticated). Use when asked to "test X in staging", "check that X's page/admin panel works", or "hit X's API endpoint". Invoke as /test-apps [app] [staging|local] [path].
---

# Test apps — Playwright for web UI, curl for JSON APIs

**Type:** procedure (drives a browser or curl against a deployed app).

Two different test surfaces, routed per app. **Don't use curl on an HTML/session-cookie app** (you'd
have to hand-scrape CSRF tokens and cookies) **and don't spin up a browser for a JSON API** (curl is
faster and scriptable). This skill tells you which one each app needs and where the actual test
tooling/credentials already live — most of it is **repo-local**, not duplicated here.

## Routing table

Fill this in for your organization. Example rows show the typical surfaces:

| App | Surface | Tool | Where |
|---|---|---|---|
| **web** | SPA dashboard | Playwright | `web/<frontend-dir>/e2e/` (existing suite) |
| **api** | Admin panel (`/admin`) | Playwright | `api/.claude/skills/navigate-web` |
| **api** | JSON API (JWT) | curl | `api/.claude/skills/test-api` |
| **<app>** | Web app (SSO session) | Playwright | no ready-made suite — recipe below |
| **worker** | JSON API (API key) | curl | recipe below |

---

## SPA with an existing Playwright suite

If a real e2e suite already exists (e.g. `web/<frontend-dir>/e2e/`), don't write a new script —
run it:

```bash
cd <path-to>/web/<frontend-dir>
npm run e2e            # full suite
npm run e2e:routes     # e.g. a spec that smoke-tests every route for errors
npm run e2e:active     # e.g. a subset of the most-used specs
```

Typical setup worth knowing (check the repo's own config):

- `playwright.config.ts` boots the dev server on `localhost:<port>` (`webServer`,
  `reuseExistingServer: true`) and points it at the **staging** API — staging data is what the specs
  assert against.
- A `global-setup.ts` logs in once via `E2E_EMAIL` / `E2E_PASSWORD` (read from the repo-root
  `.env`) and caches the session in `e2e/auth.json` for every spec. If those vars are missing it
  should throw immediately — don't hardcode credentials into a spec instead of fixing `.env`.
- To add a new check, add a `*.spec.ts` under `e2e/` following the existing ones rather than
  inventing a different pattern.

---

## Server-rendered admin panel (Playwright)

Don't write this from scratch if the repo already has repo-local skills for it — **use them**. A
good setup has two:

- **`/navigate-web`** (e.g. `api/.claude/skills/navigate-web`) — drives
  `https://<staging-api-host>/admin` in headless Chromium via a helper (e.g. `auth.mjs`'s
  `withStaging(fn)`). It handles HTTP Basic auth, any extra super-admin tier (e.g. an
  `?access_key=` parameter), per-form CSRF, and Turbo confirm dialogs for you — this is *why* it's
  Playwright and not curl for an admin panel.
- **`/test-staging-dataset`** (same repo) — test-user env vars (e.g. `AGENT_TEST_ADMIN_EMAIL` /
  `_PASSWORD` / `_ACCESS_KEY` from that repo's `.env`) plus known-good staging records (e.g. a real
  customer, order, invoice, user, documents) to use as test fixtures instead of placeholder IDs.

If the repo has neither, it is worth creating them rather than writing one-off scripts.

**Staging only** — never point `navigate-web` at production.

---

## JSON API with JWT login (curl)

If the repo has a **`/test-api`** skill (e.g. `api/.claude/skills/test-api`), use it. The usual
shape: log in via `POST /login` with the test-user email/password to get a JWT, then call endpoints
with `Authorization: Bearer <jwt>`. A sensible no-arg default is a whoami (`GET /profile`) + a
paginated index (e.g. `GET /orders?page=1`). Target defaults to local
(`BASE_URL=http://127.0.0.1:3000`); set `BASE_URL` to a staging/production host to test that
environment instead (read-only by default — confirm before any `POST`/`PUT`/`DELETE` against a
remote environment).

> **Note on paths:** docs and wiki notes drift. If the wiki shows an `/api/v1` base but the routes
> file (e.g. `config/routes.rb`) shows **top-level** routes (`POST /login`, `GET /orders`), trust
> the routes file, and correct the stale note next time it is touched.

---

## SSO-protected web app (Playwright — no ready-made suite)

An app that is a **first-party SSO client** (see the `/oauth` skill if you have one) has no local
login form, so testing it means following the SSO redirect handshake in a real browser:

```js
import { chromium } from 'playwright';

const browser = await chromium.launch();
const page = await browser.newPage();

await page.goto('https://<staging-app-host>/login');
// → redirects to https://<staging-sso-host>/sso/login?redirect_uri=...
await page.fill('input[name="email"]', EMAIL);
await page.fill('input[name="password"]', PASSWORD);
await page.click('button[type="submit"]');
// → 302 back to /sso/callback?token=<JWT>, session cookie set, lands on the app root
await page.waitForURL('**/<staging-app-host>/**');
console.log(await page.title());

await browser.close();
```

Pick the routes worth exercising once logged in from the routes file (e.g. `GET /`, list pages,
a document-generation action). Role-gated routes (e.g. `PATCH`/`DELETE /resources/:id` restricted
to admin roles) should return 403 for a lower-role test user — that is a useful check too.

**Verify the test user first:** an admin-panel HTTP-Basic user is a different auth path from a
regular SSO/JWT login. Confirm the credentials actually authenticate via `/sso/login` before
relying on them, and record whichever credentials do work.

Staging only, for the same reason as `/navigate-web` — this drives a real login against a shared
staging environment.

---

## JSON API with API-key auth (curl)

Example: a FastAPI service with routes mounted at **`/v1/...`** directly off the app root (no
`/api` prefix), authenticated by an `X-API-Key` header (a `validate_api_key` dependency), not a
bearer JWT. Check the router prefixes and auth dependency in the code before assuming.

Get a key one of two ways:

1. **Self-service temp key** — if the service exposes an unauthenticated token endpoint, good for
   a quick smoke test:
   ```bash
   BASE="https://<staging-api-host>"
   KEY=$(curl -s -X POST "$BASE/v1/auth/token" \
     -H 'Content-Type: application/json' \
     -d '{"service_identifier":"claude-test"}' \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)["api_key"])')
   ```
2. **A real service's static key** — decrypt the env file for the target environment (see
   `/credentials`, strategy `sops`) and read the relevant `*_API_KEY` entry:
   ```bash
   sops --decrypt <app>/.env.staging.enc | grep API_KEY
   ```

Then call endpoints with `X-API-Key: $KEY`:

```bash
curl -s -D - "$BASE/v1/health" -H "X-API-Key: $KEY" -w '\nHTTP %{http_code}\n'
curl -s -D - "$BASE/v1/reports/<customer_id>" -H "X-API-Key: $KEY" -w '\nHTTP %{http_code}\n'
curl -s -D - -X POST "$BASE/v1/<resource>" -H "X-API-Key: $KEY" \
  -H 'Content-Type: application/json' -d '{...}' -w '\nHTTP %{http_code}\n'
```

FastAPI serves interactive docs at `/docs` (Swagger) and the raw spec at `/openapi.json` — pull
real request shapes/params from there rather than guessing a body. A public health endpoint
(e.g. `GET /v1/health`) is the cheapest true smoke test; protected routers require the API-key
header.

---

## Rules

- **Read the hosts from your team wiki, don't hardcode URLs** — e.g. a per-app note whose
  front-matter lists the `staging` / `production` hosts.
- **Staging by default.** Never point a mutating request (`POST`/`PUT`/`DELETE`, or any admin-panel
  action with a Turbo confirm) at **production** without the user explicitly asking for it in this
  turn — same bar as a manual production deploy in `/ci-cd`.
- **Never print credentials or tokens** in output — read them from `.env` / sops / the wiki note's
  credentials section and use them directly, don't echo.
- Prefer the app's own repo-local skill/suite when one exists (`navigate-web`, `test-api`,
  `test-staging-dataset`, an existing e2e suite) over writing a new ad-hoc script — they already
  encode the app's auth quirks (CSRF, Turbo confirms, pagination headers, JWT vs API-key vs SSO
  cookie).
- This skill **observes** behavior — report failures, don't patch app code to make a check pass.
- If you discover a repo now has (or lost) a Playwright suite, or an env var / route in here is
  stale, update **this file** — it's the cross-repo index; the repo-local skills are the source of
  truth for their own app.
