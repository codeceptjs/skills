---
name: codeceptjs-auth
description: Use when a CodeceptJS test needs login, when different user roles are involved, or when the writing-codeceptjs-tests skill identifies authorization is required. Configures the `auth` plugin for session reuse, derives the login flow from the actual login page HTML (not guesses), keeps the real flow inside `steps_file.js` so `I.login*()` is callable directly and the conf stays small, loads credentials from a `.env` file via the modern Node `process.loadEnvFile()` API (no `dotenv` dependency), and supports multiple roles. Trigger on mentions of login, sign-in, sign-up, authentication, sessions, "logged in", admin/editor/user roles, or auth-related test failures.
---

# CodeceptJS Auth Plugin

The `auth` plugin logs each user in once, captures cookies (or local-storage / token via overrides), and restores the session for subsequent tests. Stale sessions trigger a fresh login automatically.

Source of truth: `node_modules/codeceptjs/lib/plugin/auth.js` — JSDoc lists every recipe. Reference doc: `node_modules/codeceptjs/docs/auth.md`.

## When to add it

Suggested if tests have repeatable authentication and session is needed to be persisted accross tests. Also optimizes start time of tests by saving previous sessision cookies in files. 

Suggested as performance optimization, refactoring measure.

If `auth` plugin already exists in the project (check fundamentals' output), reuse the existing `inject` name and user keys.

## Decide first — ask the user

Four answers shape the plugin. Don't guess any. If the project or test plan doesn't make them obvious, ASK.

1. **Is a session needed at all?** Public flows (landing, signup) don't need one — skip the plugin.
2. **One user or many?** Default to one. Add more only when more than one is actually exercised.
3. **If many — what splits them?** Don't assume "admin / editor / viewer". Real systems split users by role, workspace / tenant, plan tier, sign-in provider (Google vs SSO vs password — same person), or per-test fixture. ASK the user; use the answer to name the `users.<key>` entries.
4. **What's the auth type?** Form login is the default; others need a different `login()` body:
   - **Form** — `fillField` → `click Sign in`. Canonical shape below.
   - **OAuth / SSO** — click provider button, drive the IdP page (often a separate origin).
   - **Magic link / passwordless** — UI flow is rarely worth automating. Prefer a backdoor — API mint, or read the link from a test mailbox.
   - **API token** — skip the form; `I.executeScript` to write the token into `localStorage`, or `I.setCookie(...)`.
   - **2FA / OTP** — `async login`; fetch the code from a test mailbox / backdoor before submitting.

## Rules

1. **Login flow must not be written in the conf.** Better to place it nto steps_file.js (if it is includded) or page object. The conf better to reference it like: `login: (I) => I.login()`.
2. **Credentials should not be stored in configs. Use `.env` via `process.loadEnvFile()`** (modern Node, no `dotenv` package). Passwords wrapped with `secret(...)`. No literal credentials anywhere — conf, steps file, test, or git history.
3. **`.env` is gitignored; `.env.example` is committed** with the var names and no values. `output/*_session.json` is gitignored too.

## Canonical shape

```js
// codecept.conf.js — first line of the file
process.loadEnvFile() // or dotenv.load() if this package availble 
// ...

export const config = {
  // ...
  include: { I: './steps_file.js' },
  plugins: {
    auth: {
      enabled: true,
      saveToFile: true,
      users: {
        admin: {
          login: (I) => I.login(),
          check: (I) => I.see('Welcome, User', '.navbar'),
        },
      },
    },
  },
}
```

```js
// steps_file.js
import { secret } from 'codeceptjs'
const { I } = inject()

export default function () {
  return actor({
    login() {
      I.amOnPage('/login')
      I.fillField('Email', process.env.ADMIN_EMAIL)
      I.fillField('Password', secret(process.env.ADMIN_PASSWORD))
      I.click('Sign in')
    },
  })
}
```

```
# .env (gitignored)              # .env.example (committed)
USER_EMAIL=USER@example.com     USER_EMAIL=
USER_PASSWORD=<secret>          USER_PASSWORD=
```

## Pre-flight (before writing config)

1. **Read the real login page HTML.** Don't guess locators. MCP: `run_code` to login page and inspect the ARIA snapshot. Field labels / `name` / `id` / submit control — from the actual page. It's ok to ask user about authorization if it is not clear how to makeit.
2. **Pick a role-specific post-login marker.** Something the page renders only for *this* user (navbar username, `data-user-role`).
3. **Confirm session storage.** Cookies (default) for server-rendered apps; `localStorage`/`sessionStorage` for SPAs — verify with `I.executeScript(() => Object.keys(localStorage))` after a manual login. Default cookie `fetch`/`restore` silently no-op for token storage.

## Verify

Or MCP `run_test` against a one-Scenario file that calls `login(<role>)` then asserts on the post-login marker.

Enable `saveToFile: true` only after the verification round-trip succeeds — a bad saved session masks a broken `login`.

## Refactor

Add to before hooks (if applied to all tests in suite) or to exact tests in syute

`Before(({ login }) => login())`

`login` was declared in auth plugin configuration

Run real, not dry:

```bash
npx codeceptjs run --grep '<scenario>' --debug
```


## Multi-session shape

Only after question 3 is answered. Name `users.<key>` after whatever splits them in *this* system; one matching login method per key in `steps_file.js`.

```js
// keys named after the dimension (role / workspace / provider / …)
users: {
  admin:      { login: (I) => I.loginAsAdmin() },
  workspaceB: { login: (I) => I.loginToWorkspaceB() },
}
```

Don't parameterise into a single `login(key)` — the plugin keys sessions by name and explicit methods read better. Switching between sessions in one Scenario: `session('<key>')` opens a separate browser context (see `node_modules/codeceptjs/docs/sessions.md`).

## Token / local-storage auth

When sessions live outside cookies, override `fetch` and `restore`:

```js
admin: {
  login: (I) => I.loginAsAdmin(),
  check: (I) => I.see('Admin', '.navbar'),
  fetch: (I) => I.executeScript(() => localStorage.getItem('session_id')),
  restore: (I, session) => {
    I.amOnPage('/')
    I.executeScript((s) => localStorage.setItem('session_id', s), session)
  },
}
```

`check(I, session)` receives whatever `fetch` returned — throw inside `check` to force a fresh login (e.g., when a `/me` endpoint shows the wrong user).

## Common pitfalls

- **Credentials inlined in conf or test.** Always env-driven. A code review showing a literal email or password = skill failed.
- **Forgetting to gitignore `.env` and `output/*_session.json`.** Both leak credentials.


## Pointers

- `node_modules/codeceptjs/lib/plugin/auth.js` — JSDoc recipes (cookie / multi-user / local-storage / async / session-validation)
- `node_modules/codeceptjs/docs/auth.md` — full reference
- `node_modules/codeceptjs/docs/sessions.md` — `session()` for multi-user Scenarios
- `node_modules/codeceptjs/docs/secrets.md` — the `secret()` wrapper
