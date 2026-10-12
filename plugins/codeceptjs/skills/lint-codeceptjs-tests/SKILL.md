---
name: lint-codeceptjs-tests
description: >
  Use when a CodeceptJS 4 suite needs a static quality gate — before committing
  tests, reviewing a PR that touches tests, hardening CI, or on direct asks:
  "lint the tests", "find I.wait / setTimeout sleeps", "leftover .only or
  pause()", "missing await on grab", "passwords printed in logs", "why are
  tests slow/flaky". Runs `@codeceptjs/lint` — an AST-based linter that needs
  no CodeceptJS install and no browser.
---

# Lint CodeceptJS 4 Tests

`@codeceptjs/lint` reads tests, page objects and helpers as an AST and reports the smells that make suites slow, flaky or leaky — fixed sleeps, `if` branching, raw `fetch`, missing `await` on grabbers, plain-text credentials, leftover `.only`/`pause()`. One finding per line; exit codes `0` / `1` (errors found) / `2` (bad input, unparsable file) drop straight into CI.

## Commands

```bash
npx @codeceptjs/lint                          # files from codecept.conf.js in cwd
npx @codeceptjs/lint tests/ pages/login.js    # explicit paths (no config needed)
npx @codeceptjs/lint -r no-pause -r no-only   # only these rules (or --rules a,b)
npx @codeceptjs/lint rules                    # list rules + defaults
npx @codeceptjs/lint --json                   # file, line, column, rule, level, message, source
npx @codeceptjs/lint -c e2e/codecept.conf.js  # config in another directory
```

Without paths it reads `codecept.conf.js|ts|cjs|mjs` and lints everything matched by `tests`, local `include` files (page objects, steps file) and helpers loaded via `require`. TypeScript is parsed with Node's type stripping (Node 22.13+); `enum`/`namespace` report as parse errors. A TS config importing `./x.js` for an `x.ts` file (tsx convention) loads when the project has `tsx` installed.

## Rules and the fix for each

| Rule | Default | Detects → fix |
|---|---|---|
| `no-fixed-wait` | error | `I.wait(5)` → `I.waitForElement` / `I.waitForText` / `I.see` |
| `no-sleep` | error | bare and `window.`/`globalThis.` `setTimeout`, incl. `await new Promise(r => setTimeout(r, ms))` and `timers/promises`, outside helpers → condition wait |
| `no-if-in-test` | error | `if` inside a Scenario/hook → `tryTo()` from `codeceptjs/effects` |
| `no-fetch` | error | `fetch()` (incl. `window.fetch`, `globalThis.fetch`) outside helpers → REST helper `I.sendGetRequest` / `I.sendPostRequest` |
| `no-console` | warn | `console.*` outside helpers → `I.say()` (lands in the report) |
| `no-only` | error on CI, warn locally | `Scenario.only`, `Feature.only`, `Data(...).only` → remove before commit |
| `no-pause` | error on CI, warn locally | `pause()`, `this.pause()` → remove before commit |
| `no-skip` | warn | `Scenario.skip`, `Feature.skip`, `xScenario`, `xFeature` → intentional? suppress with a comment explaining why |
| `secret-credentials` | error | credential-ish `process.env.X` / `process.env['X']` or a password/token/secret/api-key locator passed to `I.*` → wrap with `secret()` |
| `await-grab` | error | `I.grab*()` assigned, destructured or passed without `await` (`await Promise.all([...])` and `.then()` chains are fine) → add `await` |
| `no-actor-in-helper` | error | `I` inside a class extending `Helper` → `this.helpers['Name']` |
| `raw-browser-in-test` | warn | `I.use*To`, `I.executeScript` in a Scenario → move into a helper or page object |

"On CI" = the `CI` env var is set — local debugging stubs (`pause()`, `.only`) stay warnings so they don't fail the lint while investigating.

## Configuring per project

Levels live in the config, next to the tests they govern:

```js
export const config = {
  tests: './*_test.js',
  lint: {
    rules: { 'no-if-in-test': 'warn', 'raw-browser-in-test': 'off' },
  },
}
```

Levels: `error`, `warn`, `off`. For a single deliberate case, suppress inline — rule id required, never disable a rule project-wide for one line:

```js
I.wait(1) // codeceptjs-lint-disable-line no-fixed-wait
```

## Workflow

1. Run `npx @codeceptjs/lint` at the project root; if no config loads (secrets, tsx-less TS config), pass paths explicitly.
2. Fix errors first (they fail CI), warnings by judgment. Triage by cost: sleeps and missing `await` are the flakiness drivers; `secret()` fixes are one-line.
3. For bulk cleanup, feed `--json` output to drive file-by-file edits — the `source` field holds the offending line (truncated to 120 chars).
4. Add the gate so it stays clean:

```yaml
# CI, before codeceptjs run
- run: npx @codeceptjs/lint
```

For agent-driven work, a Claude Code `PreToolUse` hook blocks edits that *add* a new error (pre-existing errors and warnings never block; a failing hook allows the edit):

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Write|Edit|MultiEdit", "hooks": [{ "type": "command", "command": "npx @codeceptjs/lint --hook claude" }] }
    ]
  }
}
```

## Things to avoid

- "Fixing" `await-grab` by deleting the `await` usage — the grab result is a promise, the fix is `await`, not removal.
- Replacing `I.wait` with a longer `I.wait`.
- Suppressing `no-if-in-test` when the branch is load-bearing — that's `tryTo()` or a helper's job.
- Adding the CI step before triaging existing findings — the suite goes red on day one.
- Skipping the hook's config scope check — it only lints files the config covers.

## Related skills

- `debugging-codeceptjs-tests` — `I.wait(N)` as a timing probe is fine mid-debug, lint-clean it before committing
- `refactoring-codeceptjs-tests` — bulk cleanup of a lint-red suite (extract helpers, tame locators)
- `writing-codeceptjs-tests` — write clean tests from the start
- `ci-fix-tests` — run the lint gate first so CI fixes don't reintroduce smells
