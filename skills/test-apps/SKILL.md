---
name: test-apps
description: Smoke-test an app's web UI or JSON API against staging (or local) — which tool each surface needs (Playwright for UIs, curl for JSON APIs), a table of each app's surface and its repo-local suite or skill, and minimal recipes per auth type (login form or SSO redirect in Playwright, JWT or API key with curl). Use when asked to "test X in staging", "check that X's page/admin panel works", or "hit X's API endpoint". Invoke as /test-apps [app] [staging|local] [path].
---

# test-apps — Playwright for UIs, curl for JSON APIs

- **Web UI** (SPA, server-rendered admin, SSO app) → **Playwright**. Session cookies, CSRF tokens
  and confirm dialogs are what a browser handles and curl doesn't.
- **JSON API** → **curl**. Faster, scriptable, and the response is the evidence.

## Apps

Fill this in for your organization. The last column is what to run **before** writing anything.

| App | Surface | Tool | Repo-local suite or skill |
|---|---|---|---|
| `<app>` | e.g. SPA dashboard | Playwright | e.g. `e2e/` (`npm run e2e`) |
| `<app>` | e.g. admin panel `/admin` | Playwright | e.g. `.claude/skills/navigate-web` |
| `<app>` | e.g. JSON API, JWT | curl | e.g. `.claude/skills/test-api` |
| `<app>` | e.g. JSON API, API key | curl | — (recipe below) |

**If the repo has a suite or skill, use it** — it already knows the app's auth quirks and test data.
Write an ad-hoc script only when there is none. Hosts: from the app's repo or your team wiki, never
guessed.

## Recipes (only when there's nothing in the repo)

**Playwright — login form or SSO redirect.** Credentials from the repo's `.env` or
`/credentials`, read into variables:

```js
import { chromium } from 'playwright';
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto(`${BASE}/login`);             // an SSO app redirects to the SSO host here
await page.fill('input[name="email"]', process.env.TEST_EMAIL);
await page.fill('input[name="password"]', process.env.TEST_PASSWORD);
await page.click('button[type="submit"]');
await page.waitForURL(`${BASE}/**`);          // back on the app, session set
await page.goto(`${BASE}/<path>`);
console.log(page.url(), await page.title());
await browser.close();
```

**curl — JWT.** Log in, keep the token in a variable, call with a bearer header:

```bash
TOKEN=$(curl -s -X POST "$BASE/login" -H 'Content-Type: application/json' \
  -d "{\"email\":\"$TEST_EMAIL\",\"password\":\"$TEST_PASSWORD\"}" | jq -r '.token')
curl -s "$BASE/<path>" -H "Authorization: Bearer $TOKEN" -w '\nHTTP %{http_code}\n'
```

**curl — API key.** Load the key from the secrets store (`/credentials`) into `$API_KEY` without
printing it:

```bash
curl -s "$BASE/<path>" -H "X-API-Key: $API_KEY" -w '\nHTTP %{http_code}\n'
```

The login path, token field and header name vary per app: check them in the code (routes file,
auth middleware/dependency), not in the wiki. Request shapes come from the routes file or the
app's OpenAPI spec (`/openapi.json`, `/docs`) — don't guess a body.

Useful checks: a health endpoint first (cheapest smoke test); a role-gated route with a lower-role
user should return 403.

## Rules

- **Staging by default.** Local only if asked. Production only if the user asks in this turn.
- **Read-only by default.** Confirm before any `POST`/`PUT`/`PATCH`/`DELETE` or admin action against
  a remote environment.
- **Never print credentials or tokens** — keep them in variables; don't `echo`, `grep` or log them,
  and don't write them to files or notes.
- **Observe, don't fix.** Report what failed (URL, status, screenshot or response); don't patch app
  code to make a check pass.
