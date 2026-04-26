---
name: codeceptjs-auth
description: Use when a CodeceptJS test needs login, when different user roles are involved, or when the writing-codeceptjs-tests skill identifies authorization is required. Configures the auth plugin for session reuse, derives the login function from the actual login page HTML (not guesses), loads credentials only from env vars, and supports multiple roles. Trigger on mentions of login, sign-in, sign-up, authentication, sessions, "logged in", admin/editor/user roles, or auth-related test failures.
---

# CodeceptJS Auth Plugin

The `auth` plugin logs each user in once, captures cookies (or local-storage / token via overrides), and restores the session for subsequent tests. If the session is stale, it re-logs-in automatically. Use it instead of repeating UI login in every `Before` hook.

Source of truth: `node_modules/codeceptjs/lib/plugin/auth.js` — the file's JSDoc lists every supported recipe.

## When to add it

- Login is slow and runs more than once across the suite.
- Two or more roles need to be exercised (admin, editor, regular user).
- Local development should reuse a session across runs.
- Tests fail because they navigate as anonymous users to protected pages.

If `auth` is already in the project (check the config-reader output), use the existing `inject` name and user keys; do not duplicate them.

## What to learn before writing config

1. **Real login page HTML.** Visit the login URL and read the markup. Identify field labels / `name` / `id` and the submit control. Do not guess locators. Prefer semantic strategies (visible label text, ARIA role+name, `name` attribute) over CSS classes that drift. If MCP is available, `run_code` to `I.amOnPage('/login')` and inspect the ARIA snapshot; otherwise open the page manually or read a captured trace.

2. **What proves the user is signed in.** Find something the post-login page renders that's specific to the role — navbar username, a "Sign out" link, a `data-user-role` attribute. The `check` function asserts on this. Pick something role-specific if you'll have multiple users.

3. **How the session is stored.** Cookies (default) work for most server-rendered apps. For SPA / token apps, the session lives in `localStorage` or `sessionStorage`; you'll override `fetch` and `restore` to use `executeScript`. Confirm by inspecting the browser after a manual login — `I.grabCookie()` and `I.executeScript(() => Object.keys(localStorage))`.

4. **Credential source — env vars only.** Use `process.env.<ROLE>_EMAIL`, `process.env.<ROLE>_PASSWORD`, or a token var. Document them in `.env.example` (without values). Wrap the password with `secret(...)` from `codeceptjs` so it never appears in step output, console logs, or trace artifacts. Never inline a credential, even temporarily.

5. **Verify the flow works.** Before declaring the skill done, run a single Scenario that calls `login(<role>)` followed by `I.see(<role-specific element>)`. If MCP is available, use `run_code` for a faster turnaround. A passing config without verification is not a finished config.

## What goes in `plugins.auth`

Inside `codecept.conf.{js,ts}`:

- `enabled: true`
- `inject` — name exposed to Scenarios (default `login`). Use a different name like `loginAs` if it collides.
- `saveToFile: true` reuses sessions across local runs by writing `output/<role>_session.json`; gitignore that path.
- `users` — one key per role. Each user provides `login` and `check`; `fetch` and `restore` only when overriding cookie defaults.

The `login` function fills the form using the locators you confirmed from the real page; passwords go through `secret()`. The `check` function navigates somewhere protected and asserts on the role-specific marker. That's it — every other field has a sane default.

## Using it in tests

- Most common: `Before(({ login }) => login('admin'))` at the top of a Feature.
- Per Scenario: inject `login` and call `login('editor')` inside the test body.
- Async login (CSRF, OTP, awaits inside `login`): declare the user's `login` as `async` and `await login('admin')` in the test.

## Multiple roles

Add more keys under `users`. Each role is a fresh saved session. Switching between them in the same test means combining `auth` with `session()` from `codeceptjs/docs/sessions.md` — `session('admin')` runs in a separate browser context, so two roles can interact in one Scenario without colliding.

## Token / local-storage auth

When the app stores the session outside cookies, override two functions:

- `fetch` returns the token (`I.executeScript(() => localStorage.getItem('session_id'))`).
- `restore` opens a page (so the origin is correct) and writes the token back (`I.executeScript((t) => localStorage.setItem('session_id', t), session)`).

Default cookie behaviour will silently no-op for SPA token storage; override or sessions won't persist.

## Session validation

`check(I, session)` receives the saved session as a second arg. If you can hit a `/me` endpoint or read a profile attribute, throw inside `check` when it doesn't match the expected role. CodeceptJS treats the throw as a stale session and triggers a fresh `login`.

## Workflow

1. Run the **codeceptjs-fundamentals** skill — confirm whether `auth` already exists, what the helper is, and which env vars the project conventions use.
2. Read the login page HTML; pick locators for email / password / submit. Pick a role-specific post-login marker.
3. Add or extend `plugins.auth` in the config. Credentials from env vars; passwords through `secret()`.
4. Add `Before(({ login }) => login(<role>))` to the affected Feature, or inject `login` per Scenario.
5. Add the new env var names to `.env.example`.
6. Run one Scenario with `--debug` (or via MCP `run_code`) to confirm login + `check` succeed end-to-end. Only then enable `saveToFile`.

## Pointers

- `node_modules/codeceptjs/lib/plugin/auth.js` — JSDoc with cookie / multi-user / local-storage / async / session-validation recipes
- `node_modules/codeceptjs/docs/sessions.md` — `session()` for multi-user Scenarios
- `node_modules/codeceptjs/docs/secrets.md` — the `secret()` wrapper
