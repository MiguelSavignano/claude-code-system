---
name: test-apps
description: Three ways to test an app against staging (or local) — Playwright to drive the real UI (login form, SSO redirect, admin panels with CSRF and confirm dialogs); curl straight to the API when you can authenticate directly (login endpoint returning a JWT, API key, basic auth); or find what already mints a token (repo-local skill, bin/scripts, package.json/Makefile/rake/manage.py task, team CLI, MCP tool, e2e auth setup, a console in a pod) and call the REST API with it. Includes a decision guide, minimal recipes, where request shapes come from, useful checks and the safety rules. Use when asked to "test X in staging", "check that X's page works", or "hit X's API endpoint". Invoke as /test-apps [app] [staging|local] [path].
argument-hint: "[app] [staging|local] [path]"
---

# test-apps — three ways to test an app

| The check is about… | Way |
|---|---|
| A page: rendering, a form, an admin action, an SSO login | **1. Playwright** |
| A JSON API you can log into directly (login endpoint, API key, basic auth) | **2. curl** |
| A JSON API whose login is hard to script (SSO, MFA, OAuth), or a token tool already exists | **3. Find the token, then curl** |

Before any of them: if the repo has an e2e suite or a repo-local skill (`ls .claude/skills`), use
it — it knows the app's auth quirks and test data. Hosts (`$BASE_URL`) come from the app repo or
the team wiki, never guessed. Credentials come from `/credentials` or the repo's `.env`
(`set -a; . ./.env; set +a`), loaded into variables, never echoed.

## 1. Playwright — drive the real UI

A browser handles cookies, CSRF tokens, redirects and confirm dialogs; curl doesn't. Prefer the
repo's suite (`npx playwright test`). Ad hoc: save as `check.mjs` (top-level await needs ESM), run
`node check.mjs` where `playwright` is installed. Selectors are examples: read the form.
```js
import { chromium } from 'playwright';
const BASE = process.env.BASE_URL;
const browser = await chromium.launch(), page = await browser.newPage();
page.on('dialog', d => d.dismiss());               // confirm dialogs: accept only if approved
await page.goto(`${BASE}/login`);                  // an SSO app redirects to the SSO host here
await page.fill('input[name="email"]', process.env.TEST_EMAIL);
await page.fill('input[name="password"]', process.env.TEST_PASSWORD);
await page.click('button[type="submit"]');
// not waitForURL(`${BASE}/**`): it matches the login page and resolves at once
await page.waitForURL(u => u.origin === new URL(BASE).origin && !u.pathname.startsWith('/login'));
// or: await page.waitForSelector('<logged-in-only element>');
await page.goto(`${BASE}/<path>`);
console.log(page.url(), await page.title());
await page.screenshot({ path: 'check.png' });
await browser.close();
```

## 2. curl — authenticate directly

The login path, token field and header name vary per app: check the code (routes file, auth
middleware/dependency), not the wiki.
```bash
# JWT: jq builds the body, -f fails on 4xx/5xx, the check avoids "Bearer null"
TOKEN=$(jq -n --arg e "$TEST_EMAIL" --arg p "$TEST_PASSWORD" '{email:$e,password:$p}' |
  curl -sf -X POST "$BASE_URL/login" -H 'Content-Type: application/json' -d @- | jq -r '.token')
if [ -z "$TOKEN" ] || [ "$TOKEN" = null ]; then echo 'login failed: no token'
else curl -s "$BASE_URL/<path>" -H "Authorization: Bearer $TOKEN" -w '\nHTTP %{http_code}\n'; fi

curl -s "$BASE_URL/<path>" -H "X-API-Key: $API_KEY" -w '\nHTTP %{http_code}\n'         # API key
curl -s "$BASE_URL/<path>" -u "$API_USER:$API_PASSWORD" -w '\nHTTP %{http_code}\n'     # basic auth
```

## 3. Find what mints the token, then call the REST API

Before scripting a login, look for what already gives you one (repo-local skill: `ls .claude/skills`):
```bash
grep -rniE 'token|jwt|login' bin scripts Makefile package.json lib/tasks .claude 2>/dev/null | head -40
bin/rails -T 2>/dev/null | grep -iE 'token|jwt|auth'           # rake tasks
python manage.py help 2>/dev/null | grep -iE 'token|jwt|auth'  # Django commands
grep -rlE 'storageState' playwright.config.* e2e tests 2>/dev/null   # e2e auth setup
```

Also the team's CLI (`<cli> --help`: `login`/`token`) and MCP tools that return a token. Capture
the output in a variable, check it, then call as in way 2:
```bash
TOKEN=$(bin/<token-script> 2>/dev/null)                        # or the task / CLI / MCP result
TOKEN=$(jq -r '.origins[].localStorage[] | select(.name=="<key>") | .value' <storage-state>.json)
[ -n "$TOKEN" ] && [ "$TOKEN" != null ] || echo 'no token'
```

A cached storage state can be expired: a 401 means re-run the suite's auth setup. Last resort: a
console command in an app pod via `/kubectl` (production needs the user's confirmation), e.g.
`TOKEN=$(kubectl exec "$POD" -n $NS -- bundle exec rails runner 'print <token call>')`, calling
the same code the login endpoint uses — find it in the code, don't invent it.

## Request shapes

Don't guess a body. Read the routes file (`config/routes.rb`, `urls.py`, the router module) and
the controller/serializer, or the app's OpenAPI spec when it serves one
(`curl -sf "$BASE_URL/openapi.json" | jq '.paths | keys'`; interactive docs at `/docs`).

## Useful checks

- **Health endpoint first** (whichever the app defines) — the cheapest smoke test.
- **Role gate:** a lower-role user on a role-gated route must get **403** (401 = login failed).
- Report evidence: URL, HTTP status, a response excerpt or screenshot (no tokens in it).

## Rules

- **Staging by default.** Local only if asked. Production only if the user asks in this turn.
- **Read-only by default.** Confirm before any `POST`/`PUT`/`PATCH`/`DELETE` or admin action against
  a remote environment.
- **Never print or write credentials or tokens** — keep them in shell variables; don't `echo`,
  log or save them to files or notes.
- **Observe, don't fix.** Report what failed; don't patch app code to make a check pass.
