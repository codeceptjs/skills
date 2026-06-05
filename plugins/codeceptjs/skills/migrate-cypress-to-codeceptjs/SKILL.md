---
name: migrate-cypress-to-codeceptjs
description: "Port a Cypress test suite to CodeceptJS 4. Trigger when the project contains `cypress.config.{js,ts,mjs}`, a `cypress/` directory (`cypress/e2e/**/*.cy.{js,ts}`, `cypress/support/{commands,e2e}.{js,ts}`, `cypress/fixtures/`, `cypress/plugins/`, `cypress/component/`), `cypress` in `devDependencies`, or test code that calls `cy.*` (`cy.visit`, `cy.get`, `cy.contains`, `cy.session`, `cy.intercept`, `cy.request`, `cy.task`, `cy.fixture`, `cy.origin`, `cy.mount`), `Cypress.Commands.add(...)`, or `Cypress.env(...)`. Walks the port end-to-end: inventory shared logic (custom commands, ad-hoc page-object modules, shared selectors, fixtures, hooks), install CodeceptJS with the Playwright helper alongside Cypress, port the config, split `Cypress.Commands.add` into two custom helpers — `WebExtra` for browser-driven commands (Playwright `page` / `browserContext`) and `ApiExtras` for HTTP commands (REST / GraphQL helper, never `browserContext.request.*`) — port page-object-style modules to real page objects without inventing wrapper or assertion methods, convert spec files (handing off to `writing-codeceptjs-tests`), replace `cy.session` with the `auth` plugin, swap `cy.fixture` / `cy.request` / `cy.task` / `cy.intercept` for ES imports / REST helper / `ApiExtras` / `I.mockRoute`, then decommission Cypress."
---

# Migrate Cypress → CodeceptJS 4

Cypress and CodeceptJS share a goal — browser end-to-end testing — but differ in three foundational ways:

1. **Step queueing vs command chains.** CodeceptJS auto-queues every `I.*` call onto an internal recorder; tests look synchronous and `await` is only needed for grabs (`await I.grabTextFrom(...)`). There is no `.then()` chain to thread state through.
2. **Helpers, not a bundled browser.** `I.*` dispatches to a configured helper. Cypress is single-browser by design; CodeceptJS lets you pick **Playwright** (recommended for Cypress migrators — Chromium parity plus cross-browser), Puppeteer, or WebDriver, and the test code stays the same.
3. **First-class abstractions.** Page objects, multi-user `session(...)`, the `auth` plugin, and custom helpers are built in. Cypress projects accumulate ad-hoc versions of these; the migration consolidates them onto the framework's idioms.

Authoritative references: `node_modules/codeceptjs/docs/basics.md`, `locators.md`, `playwright.md`, `custom-helpers.md`, `pageobjects.md`.

## When to trigger

Any of:

- `cypress.config.{js,ts,mjs}` at the repo root.
- A `cypress/` directory with `e2e/`, `support/`, `fixtures/`, `plugins/`, or `component/` subdirs.
- `cypress` listed in `devDependencies`.
- Test code calls `cy.*` (`cy.visit`, `cy.get`, `cy.contains`, `cy.session`, `cy.intercept`, `cy.request`, `cy.task`, `cy.fixture`, `cy.origin`, `cy.mount`), uses `Cypress.Commands.add(...)`, or reads `Cypress.env(...)`.
- The user says "migrate / port / convert from Cypress".

## What does not migrate

Be honest up-front:

- **Component tests** (`cy.mount`, `cypress/component/`) — CodeceptJS is E2E only. Keep Cypress for components, or move them to Playwright Component Testing / Vitest + Testing Library.
- **`cy.intercept('POST', '/api').as('save')` → `cy.wait('@save')`** — the closest equivalent is Playwright's `I.mockRoute()` (no alias, no `cy.wait('@x')`). Anchor waits on UI outcomes (`I.waitForText('Saved')`) instead of network events.
- **Cypress Cloud / time-travel debugger** — replaced by the `aiTrace` plugin's per-step artifacts and `@testomatio/reporter` for dashboards.
- **`cy.origin()` multi-origin flows** — limited support; document the gap and plan around it.

## Workflow

Run phases in order. Commit at each boundary so any regression is bisectable.

### 1. Inventory the Cypress project

Before touching anything, build a picture. Two passes.

**Shape of the project** — grep / `wc -l` for cost predictors:

- `cypress.config.{js,ts,mjs}` — which keys are in use
- `cypress/e2e/**/*.cy.{js,ts}` — spec count
- `cypress/fixtures/` — count + filenames
- `cypress/support/{e2e,commands}.{js,ts}` — these always exist; **read in full**
- `cypress/plugins/` — legacy preprocessor / task wiring
- `cypress/component/` + `cy.mount(` — flag for the user (out of scope)
- count occurrences of `cy.intercept(`, `cy.task(`, `cy.session(`, `cy.origin(`, `Cypress.Commands.add(`, `cy.fixture(` — each maps to a known replacement pattern

**Shared logic and shared locators** — Cypress has no built-in page objects, but suites accumulate shared abstractions anyway. Find them before touching test files:

- **Custom commands** — every `Cypress.Commands.add('<name>', fn)` in `cypress/support/commands.{js,ts}`. List name → arguments → body. Almost every suite has them (`cy.login`, `cy.seedData`, `cy.dragRowTo`, …).
- **Page-object-style modules** — look in `cypress/support/`, `cypress/pages/`, `cypress/page-objects/`, `cypress/helpers/`, `cypress/objects/`, `cypress/po/`, and any `pages/` / `pageObjects/` outside the cypress directory. Recognise: modules exporting selector bundles (`{ usernameField: '#user', submitBtn: '[data-cy=submit]' }`), modules exporting methods that call `cy.*` (`login(user, pwd)`, `goToProfile()`), classes with selectors as fields.
- **Shared selector constants** — files named `selectors.{js,ts}` / `locators.{js,ts}`, or modules exporting only strings. Grep specs for repeated `cy.get('[data-cy=...]')` strings — duplicates are abstraction candidates.
- **Utility helpers** — date formatters, URL builders, API wrappers (`api.js`, `helpers.js`, `utils.js`).
- **Global hooks** — `cypress/support/e2e.{js,ts}` `beforeEach` blocks, `Cypress.on('uncaught:exception', ...)`, etc.

Produce a short inventory: every shared abstraction with its current Cypress location and planned CodeceptJS destination (see phase 4's destination table). The user reviews before any code is written.

### 2. Install CodeceptJS alongside Cypress

`npx codeceptjs init` and pick the **Playwright** helper. Do not remove Cypress yet — both run in parallel through the migration, so a half-converted suite still has green coverage.

### 3. Port the config

Map `cypress.config.{js,ts}` keys → `codecept.conf.{js,ts}`:

| Cypress | CodeceptJS 4 (`Playwright` helper) |
|---|---|
| `e2e.baseUrl` | `helpers.Playwright.url` |
| `viewportWidth` / `viewportHeight` | `helpers.Playwright.windowSize: '1280x720'` |
| `defaultCommandTimeout` | `helpers.Playwright.waitForTimeout` |
| `video` | `helpers.Playwright.video: true` |
| `screenshotOnRunFailure` | plugin `screenshot` with `on: 'fail'` |
| `retries` | top-level `retry: N` |
| `env.*` / `Cypress.env('X')` | `process.env.X` |
| `setupNodeEvents` / `cy.task` | custom helper or `bootstrap` / `teardown` |

### 4. Port shared abstractions

This is the bedrock. Do it before any spec rewrite — every spec rewrite shrinks because the verbs it needs (`I.doSmth(...)`) already exist.

**Hard rule for Cypress custom commands.** Every `Cypress.Commands.add('<name>', fn)` becomes a method on a custom helper. **Split commands across two helpers by the kind of operation** — they have different access patterns and different correct APIs:

- **`WebExtra`** (`lib/helpers/WebExtra.js`) for **browser-driven** commands — anything that needs the open page, DOM, `evaluate`, init scripts, storage, network-response waits. Reaches `this.helpers['Playwright'].page` / `.browserContext`.
- **`ApiExtras`** (`lib/helpers/ApiExtras.js`) for **pure HTTP** commands — programmatic login, seed/teardown data, CRUD against an API. Reaches `this.helpers['REST']` (or `GraphQL`). See `node_modules/codeceptjs/docs/api.md` for REST helper configuration.

One async method per Cypress command, named identically, so `cy.doSmth(arg)` → `I.doSmth(arg)`. Register both helpers under `helpers` in `codecept.conf.{js,ts}`.

**Never call `this.helpers['Playwright'].browserContext.request.*` for API work.** That bypasses the REST + `JSONResponse` stack — no step logging, no `I.seeResponseCodeIsSuccessful` assertions, no shared headers, and the same verb ends up split between helpers. If the API needs the same auth as the browser, share cookies once at the top of the config:

```js
import { setSharedCookies } from '@codeceptjs/configure'
setSharedCookies()
```

…or set `defaultHeaders` on the REST helper for token-based auth, or use `I.amBearerAuthenticated(secret(token))` per test. All three patterns are covered in `api.md`.

**WebExtra example** — browser-driven commands (here `login` drives the UI form; the API-driven variant goes to `ApiExtras` below):

```js
import Helper from '@codeceptjs/helper'
import fs from 'node:fs/promises'

export default class WebExtra extends Helper {
  async login(user, password) {
    const { page } = this.helpers['Playwright']
    await page.goto('/login')
    await page.getByLabel('Email').fill(user)
    await page.getByLabel('Password').fill(password)
    await page.getByRole('button', { name: 'Sign In' }).click()
    await page.waitForURL(/\/dashboard/)
  }

  async setLocalStorage(key, value) {
    const { page } = this.helpers['Playwright']
    await page.evaluate(([k, v]) => localStorage.setItem(k, v), [key, value])
  }

  async stubWindowOpen() {
    const { page } = this.helpers['Playwright']
    await page.addInitScript(() => {
      window.__lastOpenUrl = null
      const orig = window.open
      window.open = (url, ...rest) => {
        window.__lastOpenUrl = url
        return orig ? orig.call(window, 'about:blank', ...rest) : null
      }
    })
  }

  async writeJsonFile(filePath, data) {
    await fs.writeFile(filePath, JSON.stringify(data, null, 2))
  }
}
```

**ApiExtras example** — pure HTTP commands routed through the REST helper:

```js
import Helper from '@codeceptjs/helper'

export default class ApiExtras extends Helper {
  async loginViaApi(email, password) {
    const REST = this.helpers['REST']
    await REST.sendPostRequest('/login_ajax', { email, password, remember: false })
  }

  async seedCourse(courseData) {
    const REST = this.helpers['REST']
    const { data } = await REST.sendPostRequest('/course', courseData)
    return data
  }
}
```

**Helper code style** — applies to both:

- All `import` statements at the **top of the file**. Never `const fs = await import('node:fs/promises')` inside a method.
- Use built-in assertions (`I.seeResponseCodeIsSuccessful` for API, `I.seeElement` for browser), `ExpectHelper`, or factories from `codeceptjs/assertions` — **never** `if (cond) throw new Error('...')`. Failures must render as proper assertion errors. See `node_modules/codeceptjs/docs/assertions.md`.
- If your `WebExtra` is growing a session-cache map keyed by user name, you are reimplementing the `auth` plugin — stop and let the `auth` plugin (phase 8) handle session reuse. The helper should expose `loginViaApi` / `login`; the plugin handles caching.

Cypress code that called `cy.window().then(...)`, `cy.wrap(...)`, or imperative DOM tricks translates cleanly into `page.evaluate(...)` inside `WebExtra`. Cypress code that called `cy.request(...)` translates to `REST.sendXxxRequest(...)` inside `ApiExtras`.

**Other destinations** from the phase 1 inventory:

- **Cypress page-object-style module** → CodeceptJS **page object class** under `pages/`. **Port conservatively** — keep only the methods the original module had; do not invent new wrappers during migration. Selector bundles become `this.fields = { ... }`; methods rewrite with `const { I } = inject()` at the top, calling `I.fillField`, `I.click`, and any `I.*` verb the `WebExtra` / `ApiExtras` helpers now contribute. Register under `include` in `codecept.conf.{js,ts}` so the page object auto-injects into Scenarios.

  Page-object anti-patterns to avoid (unless the original Cypress code already had them):
  - **Assertion methods** (`checkTitle() { I.seeElement(...) }`) — page objects are action verbs (`fillForm`, `submitOrder`); let assertions live in the test.
  - **One-liner wrappers** around a single `I.click` / `I.see*` / `I.grabTextFrom` — the wrapper buys nothing over calling `I.*` from the test.
  - **Methods used by only one test** — leave the steps in the test. Page objects exist for reuse.
  - **`if (cond) throw new Error(...)`** in any method — use `I.see*`, `I.seeNumberOfElements`, `ExpectHelper`, or `codeceptjs/assertions` factories instead.

- **Shared selector constants** → fields on the relevant page object. No free-floating `selectors.js`.
- **Pure utility modules** that don't touch the browser → plain ES modules, imported where needed.
- **Global hooks** → CodeceptJS `Before` / `BeforeSuite` in tests, or `bootstrap` / `teardown` in config for one-off setup.

Sanity-check before moving on: `npx codeceptjs check -c <config>` must pass, and `npx codeceptjs list -c <config>` must show every Cypress command name as an `I.*` action contributed by `WebExtra` or `ApiExtras` — whichever owns it.

### 5. Convert spec files

One file at a time, leaning on the abstractions from phase 4. Hand off the per-spec work to the **`writing-codeceptjs-tests`** skill — it drives the live browser via MCP and verifies each step before committing.

| Cypress | CodeceptJS 4 |
|---|---|
| File `*.cy.{js,ts}` | `*_test.{js,ts}` |
| `describe('X', () => { ... })` | `Feature('X')` at top, one Feature per file |
| `it('Y', () => { ... })` | `Scenario('Y', ({ I }) => { ... })` |
| `beforeEach(() => { ... })` | `Before(({ I }) => { ... })` |
| `afterEach(() => { ... })` | `After(({ I }) => { ... })` |
| `before(...)` / `after(...)` | `BeforeSuite(...)` / `AfterSuite(...)` |
| `cy.visit('/x')` | `I.amOnPage('/x')` |
| `cy.login(u, p)` (custom command) | `I.login(u, p)` (from `WebExtra`) |

**Iteration** — in tests, page objects, and helpers, use **`for...of`** for any loop containing `I.*` calls. Never `Array.prototype.forEach`. `.forEach` swallows the iteration callback's return — an `await` inside it does not block the outer function, and the CodeceptJS recorder may queue steps out of order or finish the Scenario before the loop is done. `for...of` keeps the loop sequential and lets you add `await` later without rewriting:

```js
for (const sort of testSort) {
  I.click(locate(this.filterFormLabel).withText(sort))
}
```

```js
for (const row of await I.grabWebElements('.row')) {
  const text = await row.getText()
  I.expectNotEmpty(text)
}
```

**Dry-run as you go.** After each batch of converted specs, run:

```bash
npx codeceptjs dry-run --steps -c <config>
```

It loads every scenario, resolves every `I.*` call against the configured helpers, and prints the step list — all without launching a browser. Typos, missing imports, page objects not registered under `include`, and `I.*` verbs that don't exist on `WebExtra` / `ApiExtras` all surface here in seconds. Fix anything that fails before running a real test.

**Then run the whole batch for real.** Dry-run proves specs parse and resolve — not that they pass. As soon as a batch is dry-run clean, run it against the browser:

```bash
npx codeceptjs run --steps -c <config>
```

First real runs after a migration almost always have failures — locator drift, timing the source framework hid behind its own retry, auth/session differences, data assumptions. **This is expected; fixing it is part of the migration, not a follow-up task.** When a test fails, **invoke the `debugging-codeceptjs-tests` skill and fix it on the fly** — it breakpoints the failing step, inspects the live page via MCP, finds the working locator/wait, and commits the verified fix. Do not bulk-rewrite specs blind, and do not mask failures with `retry`. Drive every failure to a real fix before starting the next batch. A batch is "done" when it runs green, not when it dry-runs clean.

### 6. Locators

CodeceptJS priority — pick the highest that fits:

1. **Semantic strings** — button text, label, placeholder, link text: `I.click('Save')`, `I.fillField('Email', 'u@t.com')`.
2. **ARIA roles** — `I.click({ role: 'button', name: 'Sign In' })`.
3. **`locate()` builder** — `I.click(locate('button').withText('Edit').inside('tr').withText('Acme')))`.
4. **CSS / XPath** — fallback only.

Cypress users often default to `[data-cy=...]`. Keep those attributes, but enable the `customLocator` plugin so they read as `$submit` instead of `{ css: '[data-cy=submit]' }`. Full guidance in **`writing-codeceptjs-tests`** § Locators.

### 7. Actions, assertions, grabs

| Cypress | CodeceptJS 4 |
|---|---|
| `cy.get(sel).click()` | `I.click(sel)` |
| `cy.get(sel).type('x')` | `I.fillField(sel, 'x')` |
| `cy.get(sel).clear()` | `I.clearField(sel)` |
| `cy.get(sel).check()` / `.uncheck()` | `I.checkOption(sel)` / `I.uncheckOption(sel)` |
| `cy.get(sel).select('A')` | `I.selectOption(sel, 'A')` |
| `cy.get(sel).should('be.visible')` | `I.seeElement(sel)` |
| `cy.get(sel).should('have.text', 'X')` | `I.see('X', sel)` |
| `cy.get(sel).should('have.value', 'X')` | `I.seeInField(sel, 'X')` |
| `cy.get(sel).should('have.length', 5)` | `I.seeNumberOfElements(sel, 5)` |
| `cy.url().should('include', '/x')` | `I.seeInCurrentUrl('/x')` |
| `cy.get(sel).invoke('text').then(t => ...)` | `const t = await I.grabTextFrom(sel)` |
| `cy.getCookie('s')` | `const c = await I.grabCookie('s')` |

`await` only on grabs. Plain actions queue automatically.

### 8. Sessions and auth

`cy.session(id, setup, { validate })` and `cy.request`-based programmatic login → the **`auth` plugin**. Hand off to **`codeceptjs-auth`** for the setup walk-through. If phase 4 already ported `cy.login` into `WebExtra` as `I.login(...)`, the `auth` plugin's role definition just calls `I.login(...)`. For multi-user scenarios (Cypress has no native equivalent) use `session(...)` from `codeceptjs/effects`.

### 9. Fixtures, requests, tasks

| Cypress | CodeceptJS 4 |
|---|---|
| `cy.fixture('users.json')` | `import users from './fixtures/users.json' with { type: 'json' }` |
| `cy.request('POST', '/api/x', body)` | `await I.sendPostRequest('/api/x', body)` via the **REST helper**; for reusable flows wrap in the `ApiExtras` helper from phase 4 |
| `cy.task('seedDB')` | method on `ApiExtras` (if HTTP), a dedicated helper, or `bootstrap` / `teardown` |

REST helper auth: `setSharedCookies()` from `@codeceptjs/configure` shares the browser session with REST so the same user is logged in on both sides; alternatively set `defaultHeaders` for static tokens or `I.amBearerAuthenticated(secret(token))` per test. See `node_modules/codeceptjs/docs/api.md` for the full configuration surface, including `JSONResponse` assertions (`I.seeResponseCodeIsSuccessful`, `I.seeResponseContainsKeys`, `I.seeResponseMatchesJsonSchema` with Zod).

### 10. Network mocking

`cy.intercept(url, handler)` → `I.mockRoute(url, route => route.fulfill({ ... }))` (Playwright). Disable with `I.stopMockingRoute(url)`. There is no `cy.wait('@alias')` equivalent — anchor waits on UI outcomes (`I.waitForText`, `I.seeElement`) instead of network events.

### 11. Decommission Cypress

Only after every spec is ported and CI is green: delete `cypress/`, `cypress.config.*`, drop `cypress` from `devDependencies`, remove the Cypress CI jobs.

## Verify

1. `npx codeceptjs check -c <config>` — config + helper + plugin sanity.
2. `npx codeceptjs list -c <config>` — every ported Cypress command appears as an `I.*` action from `WebExtra` or `ApiExtras`; every page object's methods appear.
3. `npx codeceptjs dry-run --steps -c <config>` — every Scenario loads.
4. Full run: `npx codeceptjs run --steps -c <config>`. Failures are expected on first runs — drive each to a fix via the **`debugging-codeceptjs-tests`** skill (not `retry`, not blind rewrites). The migration is complete only when the whole converted suite is green.
5. Hand off to **`codeceptjs-run-analysis`** to inspect `output/trace_*/` artifacts (requires the `aiTrace` plugin enabled).
6. `grep -r "cy\." cypress/` — empty before deleting `cypress/`.

## Pointers

- `node_modules/codeceptjs/docs/basics.md` — `I.*` vocabulary, locators, assertions, the `await` rule
- `node_modules/codeceptjs/docs/playwright.md` — recommended helper; `mockRoute` for `cy.intercept`
- `node_modules/codeceptjs/docs/locators.md` — semantic / ARIA / `locate()`
- `node_modules/codeceptjs/docs/custom-helpers.md` — `WebExtra` / `ApiExtras` patterns (extending `Helper`, reaching `this.helpers['Playwright']` / `this.helpers['REST']`)
- `node_modules/codeceptjs/docs/api.md` — REST / GraphQL configuration, `setSharedCookies()`, `defaultHeaders`, `JSONResponse` assertions, Zod schemas
- `node_modules/codeceptjs/docs/assertions.md` — built-in `see*` assertions, `ExpectHelper`, `codeceptjs/assertions` factories (use these instead of `if (cond) throw new Error(...)`)
- `node_modules/codeceptjs/docs/pageobjects.md` — porting Cypress page-object-style modules
- `node_modules/codeceptjs/docs/data.md` — fixtures, data factories
- `node_modules/codeceptjs/docs/sessions.md`, `auth.md` — multi-user + login reuse
- `node_modules/codeceptjs/docs/effects.md` — `tryTo`, `retryTo`, `within`
- `writing-codeceptjs-tests` — per-spec rewrite playbook (drive via MCP, learn locators, commit verified steps)
- `debugging-codeceptjs-tests` — **use on every failing test from the first full run** (breakpoint, inspect live page via MCP, fix on the fly)
- `codeceptjs-auth` — replace `cy.session()` and programmatic login
- `codeceptjs-fundamentals` — run **after** migration to confirm the new setup is wired correctly
