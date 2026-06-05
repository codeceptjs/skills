---
name: migrate-protractor-to-codeceptjs
description: "Port a Protractor test suite to CodeceptJS 4. Trigger when the project contains `protractor.conf.{js,ts}`, `protractor` in `devDependencies`, `*.e2e-spec.{js,ts}` files, an `e2e/` (or `protractor/`) directory with spec files, `@types/jasmine` / `jasmine-spec-reporter` in dependencies, imports from `protractor` (`browser`, `element`, `by`, `ExpectedConditions`, `ElementFinder`, `ElementArrayFinder`), or code calling `element(by.X(...))`, `element.all(...)`, `by.addLocator(...)`, `browser.get(...)`, `browser.executeScript(...)`, `browser.wait(EC.*)`, `browser.waitForAngular(...)`, `browser.ignoreSynchronization`, `browser.params.*`, or `browser.driver.*`. Walks the port end-to-end: inventory shared logic (page objects — Protractor projects almost always have them, custom locators via `by.addLocator`, shared helpers, `onPrepare` / `onComplete` hooks, Jasmine custom matchers), install CodeceptJS with the Playwright helper alongside Protractor, port the config, split shared helpers into `WebExtra` (browser-driven via Playwright `page`) and `ApiExtras` (HTTP via REST helper, never `browserContext.request.*`), port existing page objects to CodeceptJS page objects without inventing assertion/one-liner wrappers, replace `.then(...)` promise chains with `await` only on grabs, drop `browser.waitForAngular()` / `browser.ignoreSynchronization` (CodeceptJS auto-waits), translate `element(by.X(...))` to semantic strings / ARIA / `locate()` / `{ css }`, register `by.addLocator` strategies via the `customLocator` plugin or `WebExtra`, convert specs (handing off to `writing-codeceptjs-tests`), replace `browser.params` with `process.env`, swap Jasmine `expect()` matchers for `I.see*` / `ExpectHelper` / `codeceptjs/assertions`, then decommission Protractor."
---

# Migrate Protractor → CodeceptJS 4

Protractor was end-of-lifed in April 2023. CodeceptJS 4 is a strong target: it speaks WebDriver natively (the runtime Protractor was built on) and also supports Playwright, which is faster, less flaky, and recommended for new work. The migration also retires three legacies that were already deprecated in Protractor itself: the Selenium ControlFlow, the Angular synchronization hook (`waitForAngular`), and `.then()` promise chains for queueing browser work.

Three foundational differences to internalize:

1. **No more promise chains.** Protractor's `.then()` chains queued work on the Selenium ControlFlow; CodeceptJS auto-queues `I.*` calls via an internal recorder, so tests read synchronously. `await` is only needed for grabs (`await I.grabTextFrom(...)`).
2. **Helpers, not `browser` / `driver`.** `I.*` dispatches to a configured helper. **Playwright recommended**; WebDriver is also available if the suite must keep running against a Selenium Grid — test code is identical either way.
3. **Auto-wait, not Angular-wait.** Drop `browser.waitForAngular()` and `browser.ignoreSynchronization`. The Playwright and WebDriver helpers wait on DOM and element stability, which covers Angular's render cycle without a framework-specific hook.

Authoritative references: `node_modules/codeceptjs/docs/basics.md`, `locators.md`, `playwright.md`, `webdriver.md`, `custom-helpers.md`, `pageobjects.md`.

## When to trigger

Any of:

- `protractor.conf.{js,ts}` at the repo root.
- `protractor` listed in `devDependencies` (often alongside `@types/jasmine`, `jasmine`, `jasmine-spec-reporter`).
- An `e2e/` or `protractor/` directory with spec files (commonly `*.e2e-spec.{js,ts}` or `*.spec.{js,ts}`).
- Imports from `protractor` (`browser`, `element`, `by`, `ExpectedConditions`, `ElementFinder`, `ElementArrayFinder`).
- Code calls `element(by.X(...))`, `element.all(...)`, `by.addLocator(...)`, `browser.get(...)`, `browser.executeScript(...)`, `browser.wait(EC.*)`, `browser.waitForAngular()`, `browser.ignoreSynchronization`, `browser.params.*`, or `browser.driver.*`.
- The user says "migrate / port / convert from Protractor".

## What does not migrate

Be honest up-front:

- **Selenium ControlFlow** — gone. The migration includes converting any remaining ControlFlow-style sequencing to plain `async/await`. If `SELENIUM_PROMISE_MANAGER` was already disabled in the project, this is mostly mechanical; if it was still on, audit every spec for implicit ordering.
- **`browser.waitForAngular()` / `browser.ignoreSynchronization` / `browser.waitForAngularEnabled(false)`** — drop. CodeceptJS auto-waits on DOM stability, which is what Angular needs anyway. If a step relied on Angular sync to mask a real race, it will fail loudly after migration — fix it with a specific `I.waitFor*`.
- **Angular-specific locator strategies** — `by.binding(...)`, `by.repeater(...)`, `by.model(...)`, `by.options(...)` have no built-in CodeceptJS equivalent. Replace with CSS attribute selectors (`{ css: '[ng-model="user.email"]' }`) or, if widely used, register custom locator strategies in `WebExtra` / `customLocator`.
- **Jasmine test infrastructure** — `jasmineNodeOpts`, custom matchers (`jasmine.addMatchers`), `jasmine-spec-reporter`. CodeceptJS uses Mocha; matchers translate to `ExpectHelper` / `codeceptjs/assertions`, reporters to CodeceptJS plugins (`@testomatio/reporter`, `mochawesome`, etc.).
- **`browser.params`** — replaced by plain `process.env.*`. No equivalent of Protractor's typed params object.

## Workflow

Run phases in order. Commit at each boundary so any regression is bisectable.

### 1. Inventory the Protractor project

Before touching anything, build a picture. Two passes.

**Shape of the project** — grep / `wc -l` for cost predictors:

- `protractor.conf.{js,ts}` — which keys are in use (`seleniumAddress`, `directConnect`, `capabilities`, `baseUrl`, `specs`, `params`, `onPrepare`, `onComplete`, `framework`, `jasmineNodeOpts`, `allScriptsTimeout`)
- `**/*.e2e-spec.{js,ts}` (or `**/*.spec.{js,ts}` inside an `e2e/` directory) — spec count
- imports of `protractor` — every file that uses `browser`, `element`, `by`, `ExpectedConditions`
- count occurrences of `.then(`, `by.addLocator(`, `browser.waitForAngular(`, `browser.executeScript(`, `browser.ignoreSynchronization`, `browser.params.`, `EC.` — each maps to a known replacement pattern

**Shared logic and page objects** — Protractor projects almost always have an explicit Page Objects pattern (it's the recommended idiom in Protractor's own docs). Find them before touching specs:

- **Page objects** — typically under `e2e/page-objects/`, `e2e/pages/`, `e2e/po/`, or `e2e/<feature>/<feature>.po.ts`. Modules that export classes (or plain objects) with `element(by.X(...))` properties (lazy `ElementFinder` references) and methods that drive interactions. These are first-class abstractions — port them straight across to CodeceptJS page objects.
- **Custom locators** — every `by.addLocator('<name>', fn)`, usually in `onPrepare` or a helpers file. Each becomes a custom strategy in `WebExtra` or a row in the `customLocator` plugin config.
- **Helper modules** — under `e2e/helpers/` or `e2e/utils/`. UI helpers (DOM tricks, executeScript) go to `WebExtra`; HTTP helpers (programmatic login, seed/teardown data) go to `ApiExtras`.
- **`onPrepare` / `onComplete` hooks** — Protractor's global setup/teardown. Become `bootstrap()` / `teardown()` in `codecept.conf.{js,ts}` for one-off setup, or `Before` / `BeforeSuite` in a base spec for per-suite setup.
- **Jasmine custom matchers** — `jasmine.addMatchers({ ... })` definitions become reusable assertions inside a custom helper using `codeceptjs/assertions` factories.

Produce a short inventory: every shared abstraction with its current location and planned CodeceptJS destination. The user reviews before any code is written.

### 2. Install CodeceptJS alongside Protractor

`npx codeceptjs init` and pick the **Playwright** helper (modern, faster, less flaky than Selenium). Pick **WebDriver** instead only if the team must keep running against a Selenium Grid — the test code is identical either way. Do not remove Protractor yet — both run in parallel through the migration, so a half-converted suite still has green coverage.

### 3. Port the config

Map `protractor.conf.{js,ts}` keys → `codecept.conf.{js,ts}`:

| Protractor | CodeceptJS 4 (`Playwright` helper) |
|---|---|
| `baseUrl` | `helpers.Playwright.url` |
| `capabilities.browserName` (`chrome` / `firefox`) | `helpers.Playwright.browser` (`chromium` / `firefox` / `webkit`) |
| `capabilities.chromeOptions.args` | `helpers.Playwright.chromium.args` / `launchOptions.args` |
| `directConnect: true` | drop — Playwright manages the browser |
| `seleniumAddress` | drop (Playwright), or `helpers.WebDriver.host` / `port` (WebDriver helper) |
| `specs: ['./e2e/**/*.e2e-spec.ts']` | `tests: './tests/**/*_test.{js,ts}'` |
| `allScriptsTimeout` / `defaultTimeoutInterval` | `helpers.Playwright.waitForTimeout`, top-level `timeout` |
| `params: { ... }` | `process.env.*` |
| `framework: 'jasmine'` / `jasmineNodeOpts` | drop — CodeceptJS uses Mocha |
| `onPrepare(...)` | `bootstrap()` |
| `onComplete(...)` | `teardown()` |
| `SELENIUM_PROMISE_MANAGER: false` | drop — async/await is mandatory in CodeceptJS |
| `useAllAngular2AppRoots` / `rootElement` | drop — not needed |

### 4. Port shared abstractions

This is the bedrock. Do it before any spec rewrite — every spec rewrite shrinks because the verbs it needs (`I.doSmth(...)`) already exist.

**Hard rule for shared helper code.** Every method on a Protractor helper module becomes a method on a custom CodeceptJS helper. **Split across two helpers by the kind of operation** — they have different access patterns and different correct APIs:

- **`WebExtra`** (`lib/helpers/WebExtra.js`) for **browser-driven** operations — anything that needs the open page, DOM, `evaluate`, init scripts, storage, network-response waits. This is where every `browser.executeScript(...)` call lands, as `page.evaluate(...)`. Reaches `this.helpers['Playwright'].page` / `.browserContext`.
- **`ApiExtras`** (`lib/helpers/ApiExtras.js`) for **pure HTTP** operations — programmatic login, seed/teardown data, CRUD against an API. Reaches `this.helpers['REST']` (or `GraphQL`). See `node_modules/codeceptjs/docs/api.md` for REST helper configuration.

Register both helpers under `helpers` in `codecept.conf.{js,ts}`.

**Never call `this.helpers['Playwright'].browserContext.request.*` for API work.** That bypasses the REST + `JSONResponse` stack — no step logging, no `I.seeResponseCodeIsSuccessful` assertions, no shared headers. If the API needs the same auth as the browser, share cookies once at the top of the config:

```js
import { setSharedCookies } from '@codeceptjs/configure'
setSharedCookies()
```

…or set `defaultHeaders` on the REST helper for token-based auth, or use `I.amBearerAuthenticated(secret(token))` per test. All three patterns are covered in `api.md`.

**WebExtra example** — browser-driven operations, including the canonical `browser.executeScript` → `page.evaluate` translation:

```js
import Helper from '@codeceptjs/helper'

export default class WebExtra extends Helper {
  async setLocalStorage(key, value) {
    const { page } = this.helpers['Playwright']
    await page.evaluate(([k, v]) => localStorage.setItem(k, v), [key, value])
  }

  async scrollIntoView(selector) {
    const { page } = this.helpers['Playwright']
    await page.locator(selector).scrollIntoViewIfNeeded()
  }

  async waitForAngularRouterEvents(timeout = 10000) {
    const { page } = this.helpers['Playwright']
    await page.waitForFunction(
      () => !document.querySelector('.cdk-overlay-backdrop, .mat-progress-bar'),
      { timeout },
    )
  }
}
```

**ApiExtras example** — pure HTTP operations routed through the REST helper:

```js
import Helper from '@codeceptjs/helper'

export default class ApiExtras extends Helper {
  async loginViaApi(email, password) {
    const REST = this.helpers['REST']
    await REST.sendPostRequest('/api/auth/login', { email, password })
  }

  async seedUser(user) {
    const REST = this.helpers['REST']
    const { data } = await REST.sendPostRequest('/api/users', user)
    return data
  }
}
```

**Helper code style** — applies to both:

- All `import` statements at the **top of the file**. Never `const fs = await import('node:fs/promises')` inside a method.
- Use built-in assertions (`I.seeResponseCodeIsSuccessful` for API, `I.seeElement` for browser), `ExpectHelper`, or factories from `codeceptjs/assertions` — **never** `if (cond) throw new Error('...')`. Failures must render as proper assertion errors. See `node_modules/codeceptjs/docs/assertions.md`.

**Custom locator strategies** (`by.addLocator`) — two destinations:

- **Simple attribute strategies** (translate to a CSS / XPath under the hood) → enable the **`customLocator` plugin** and configure each strategy. Once on, `I.click('$submit')` resolves the `submit` strategy.
- **Complex strategies** (run JavaScript against the DOM) → method on `WebExtra` that returns the matched element, plus a thin `I.click*` / `I.see*` wrapper if needed.

**Other destinations** from the phase 1 inventory:

- **Protractor page objects** → CodeceptJS **page object class** under `pages/`. **Port conservatively** — keep only the methods the original page object had; do not invent new wrappers during migration. `ElementFinder` properties (`get usernameField() { return element(by.id('user')) }`) become locator-string fields (`fields = { usernameField: '#user' }`); methods rewrite with `const { I } = inject()` at the top, calling `I.fillField`, `I.click`, and any `I.*` verb the `WebExtra` / `ApiExtras` helpers now contribute. Register under `include` in `codecept.conf.{js,ts}` so the page object auto-injects into Scenarios.

  Page-object anti-patterns to avoid (unless the original Protractor page object already had them):
  - **Assertion methods** (`checkTitle() { I.seeElement(...) }`) — page objects are action verbs (`fillForm`, `submitOrder`); let assertions live in the test.
  - **One-liner wrappers** around a single `I.click` / `I.see*` / `I.grabTextFrom` — the wrapper buys nothing over calling `I.*` from the test.
  - **Methods used by only one test** — leave the steps in the test.
  - **`if (cond) throw new Error(...)`** in any method — use `I.see*`, `I.seeNumberOfElements`, `ExpectHelper`, or `codeceptjs/assertions` factories instead.

- **Shared selector constants** → fields on the relevant page object. No free-floating `selectors.{js,ts}`.
- **Pure utility modules** that don't touch the browser → plain ES modules, imported where needed.
- **`onPrepare` / `onComplete`** → `bootstrap` / `teardown` in `codecept.conf.{js,ts}`, or `BeforeSuite` / `AfterSuite` if the work is per-spec.

Sanity-check before moving on: `npx codeceptjs check -c <config>` must pass, and `npx codeceptjs list -c <config>` must show every ported helper method as an `I.*` action contributed by `WebExtra` or `ApiExtras`.

### 5. Convert spec files

One file at a time, leaning on the abstractions from phase 4. Hand off the per-spec work to the **`writing-codeceptjs-tests`** skill — it drives the live browser via MCP and verifies each step before committing.

| Protractor | CodeceptJS 4 |
|---|---|
| File `*.e2e-spec.{js,ts}` | `*_test.{js,ts}` |
| `describe('X', () => { ... })` | `Feature('X')` at top, one Feature per file |
| `it('Y', () => { ... })` | `Scenario('Y', ({ I }) => { ... })` |
| `beforeEach(() => { ... })` | `Before(({ I }) => { ... })` |
| `afterEach(() => { ... })` | `After(({ I }) => { ... })` |
| `beforeAll(...)` / `afterAll(...)` | `BeforeSuite(...)` / `AfterSuite(...)` |
| `browser.get('/x')` | `I.amOnPage('/x')` |
| `await loginPage.login(u, p)` | `loginPage.login(u, p)` (no `await` on void page-object methods that wrap actions) |

**Drop `browser.waitForAngular()` / `browser.ignoreSynchronization`** outright — every call site. CodeceptJS auto-waits.

**Promise chains** — most `.then(...)` chains collapse to plain statements because the recorder queues actions. `await` only on grabs:

```js
const text = await I.grabTextFrom('.foo')
I.expectEqual(text, 'Hello')
```

**Iteration** — in tests, page objects, and helpers, use **`for...of`** for any loop containing `I.*` calls. Never `Array.prototype.forEach`. `.forEach` swallows the iteration callback's return — an `await` inside it does not block the outer function, and the recorder may queue steps out of order or finish the Scenario before the loop is done. `for...of` keeps the loop sequential and lets you add `await` later without rewriting:

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

### 6. Locator preference

CodeceptJS priority — pick the highest that fits:

1. **Semantic strings** — button text, label, placeholder, link text: `I.click('Save')`, `I.fillField('Email', 'u@t.com')`. Covers Protractor's `by.linkText`, `by.buttonText`, `by.partialButtonText`, `by.partialLinkText` cleanly.
2. **ARIA roles** — `I.click({ role: 'button', name: 'Sign In' })`. Strong default for Angular apps that ship Material / ARIA-correct components.
3. **`locate()` builder** — `I.click(locate('.row').withText('Acme').inside('table'))`. Direct equivalent of `by.cssContainingText` + element traversal chains.
4. **CSS / XPath / attribute objects** — `{ id: 'foo' }`, `{ name: 'email' }`, `{ css: '[ng-model="user.email"]' }`, `{ xpath: '//div[@id="x"]' }`. The fallback for Angular directive attributes.

| Protractor locator | CodeceptJS 4 |
|---|---|
| `by.css('.btn')` | `'.btn'` |
| `by.id('foo')` | `{ id: 'foo' }` |
| `by.name('email')` | `{ name: 'email' }` |
| `by.tagName('input')` | `{ css: 'input' }` |
| `by.linkText('Sign In')` | `'Sign In'` (semantic) |
| `by.partialLinkText('Sig')` | `'Sig'` (semantic, partial by default) |
| `by.buttonText('Submit')` | `'Submit'` (semantic) |
| `by.partialButtonText('Sub')` | `'Sub'` (semantic) |
| `by.xpath('//div[@id="x"]')` | `{ xpath: '//div[@id="x"]' }` |
| `by.cssContainingText('.row', 'Acme')` | `locate('.row').withText('Acme')` |
| `by.binding('user.name')` | `{ css: '[ng-bind="user.name"]' }` or custom locator |
| `by.model('user.email')` | `{ css: '[ng-model="user.email"]' }` |
| `by.repeater('item in items')` | `{ css: '[ng-repeat="item in items"]' }` |
| `by.options('opt for opt in options')` | `{ css: '[ng-options="opt for opt in options"]' }` |
| `by.addLocator('myLocator', fn)` | `customLocator` plugin, or method on `WebExtra` |

**Element traversal** — Protractor's chains (`element(...).element(...)`, `.all(...)`, `.first()`, `.get(N)`) collapse onto CodeceptJS's context arg, `locate()` chain, and `step.opts`:

| Protractor | CodeceptJS 4 |
|---|---|
| `element(by.X)` | the locator alone (`'.foo'`, `{ role: ... }`, etc.) |
| `element.all(by.X)` | `await I.grabWebElements(sel)` |
| `element.all(by.X).count()` (asserted) | `I.seeNumberOfElements(sel, N)` |
| `element.all(by.X).first()` | `step.opts({ elementIndex: 'first' })` (or `1`) |
| `element.all(by.X).last()` | `step.opts({ elementIndex: 'last' })` |
| `element.all(by.X).get(N)` | `step.opts({ elementIndex: N + 1 })` |
| `element(by.X).element(by.Y)` | context arg: `I.click(Y, X)` — or `locate(Y).inside(X)` |
| `element(by.X).$('.foo')` | same |

`step.opts(...)` comes from `import step from 'codeceptjs/steps'`.

### 7. Actions, assertions, grabs

| Protractor | CodeceptJS 4 |
|---|---|
| `element(sel).click()` | `I.click(sel)` |
| `element(sel).sendKeys('x')` | `I.fillField(sel, 'x')` |
| `element(sel).clear()` | `I.clearField(sel)` |
| `element(sel).submit()` | `I.click('Submit', form)` or `I.pressKey('Enter')` |
| `element(sel).getText().then(...)` | `const t = await I.grabTextFrom(sel)` |
| `element(sel).getAttribute('data-id')` | `await I.grabAttributeFrom(sel, 'data-id')` |
| `element(sel).isDisplayed()` (asserted) | `I.seeElement(sel)` |
| `element(sel).isPresent()` (asserted) | `I.seeElementInDOM(sel)` |
| `expect(element(sel).getText()).toEqual('X')` | `I.see('X', sel)` |
| `expect(element.all(sel).count()).toBe(N)` | `I.seeNumberOfElements(sel, N)` |
| `expect(browser.getCurrentUrl()).toContain('/x')` | `I.seeInCurrentUrl('/x')` |
| `browser.executeScript(fn, args)` | inside a helper: `await page.evaluate(fn, args)` |
| `browser.refresh()` | `I.refreshPage()` |
| `browser.getCurrentUrl()` | `await I.grabCurrentUrl()` |
| `browser.getTitle()` | `await I.grabTitle()` |
| `browser.sleep(N)` | `I.wait(N / 1000)` — CodeceptJS uses **seconds**; avoid raw waits in committed tests |
| `browser.wait(EC.visibilityOf(el), 5000)` | `I.waitForVisible(sel, 5)` |
| `browser.wait(EC.invisibilityOf(el), 5000)` | `I.waitForInvisible(sel, 5)` |
| `browser.wait(EC.presenceOf(el), 5000)` | `I.waitForElement(sel, 5)` |
| `browser.wait(EC.textToBePresentInElement(el, 'X'), 5000)` | `I.waitForText('X', 5, sel)` |
| `browser.wait(EC.urlContains('/x'), 5000)` | `I.waitInUrl('/x', 5)` |
| `browser.waitForAngular()` | drop |
| `browser.params.user` | `process.env.USER` |

`await` only on grabs. Plain actions queue automatically.

### 8. Sessions and auth

Protractor had no native session reuse — most projects either logged in via the UI in every `beforeEach`, or reached into `browser.driver.manage().addCookie(...)` for shortcuts. Replace both with the **`auth` plugin**. Hand off to **`codeceptjs-auth`** for the setup walk-through. If phase 4 already ported your login into `ApiExtras` as `I.loginViaApi(...)` or into `WebExtra` as `I.login(...)`, the `auth` plugin's role definition just calls it. For multi-user scenarios (Protractor had no native equivalent), use `session(...)` from `codeceptjs/effects`.

### 9. Fixtures, requests, tasks

| Protractor | CodeceptJS 4 |
|---|---|
| `require('./fixtures/users.json')` | `import users from './fixtures/users.json' with { type: 'json' }` |
| `http.request(...)` inside `onPrepare` | `await I.sendPostRequest(...)` via the **REST helper**; wrap reusable flows in the `ApiExtras` helper from phase 4 |
| `protractor-cucumber-framework` / Cucumber hooks | CodeceptJS BDD (`gherkin:steps`) or plain `bootstrap` / `teardown` |

REST helper auth: `setSharedCookies()` from `@codeceptjs/configure` shares the browser session with REST so the same user is logged in on both sides; alternatively set `defaultHeaders` for static tokens or `I.amBearerAuthenticated(secret(token))` per test. See `node_modules/codeceptjs/docs/api.md`.

### 10. Network mocking

Protractor had no built-in network interception — projects typically rolled their own via `browser.executeScript` patching `XMLHttpRequest`, or via a backend stub. For Playwright-based CodeceptJS migration: use `I.mockRoute(url, route => route.fulfill({ ... }))` instead, or `I.stopMockingRoute(url)` to disable. See `node_modules/codeceptjs/docs/playwright.md` § Mocking Network Requests.

### 11. Decommission Protractor

Only after every spec is ported and CI is green: delete `e2e/` (or whichever directory held Protractor specs), `protractor.conf.*`, drop `protractor`, `@types/jasmine`, `jasmine`, `jasmine-spec-reporter` from `devDependencies`, remove the Protractor CI jobs, and uninstall the matching Chrome / Selenium webdriver-manager binaries.

## Verify

1. `npx codeceptjs check -c <config>` — config + helper + plugin sanity.
2. `npx codeceptjs list -c <config>` — every ported helper method appears as an `I.*` action from `WebExtra` or `ApiExtras`; every page object's methods appear.
3. `npx codeceptjs dry-run --steps -c <config>` — every Scenario loads.
4. Full run: `npx codeceptjs run --steps -c <config>`. Failures are expected on first runs — drive each to a fix via the **`debugging-codeceptjs-tests`** skill (not `retry`, not blind rewrites). The migration is complete only when the whole converted suite is green.
5. Hand off to **`codeceptjs-run-analysis`** to inspect `output/trace_*/` artifacts (requires the `aiTrace` plugin enabled).
6. `grep -rE "\\bbrowser\\.|\\bby\\.|\\.then\\(|waitForAngular" e2e/` — empty before deleting `e2e/`.

## Pointers

- `node_modules/codeceptjs/docs/basics.md` — `I.*` vocabulary, locators, assertions, the `await` rule
- `node_modules/codeceptjs/docs/playwright.md` — recommended helper; `mockRoute` for any network mocking; `evaluate` for `executeScript` ports
- `node_modules/codeceptjs/docs/webdriver.md` — alternative helper if the suite stays on Selenium Grid
- `node_modules/codeceptjs/docs/locators.md` — semantic / ARIA / `locate()`, `customLocator` plugin for `by.addLocator` replacements
- `node_modules/codeceptjs/docs/custom-helpers.md` — `WebExtra` / `ApiExtras` patterns (extending `Helper`, reaching `this.helpers['Playwright']` / `this.helpers['REST']`)
- `node_modules/codeceptjs/docs/api.md` — REST / GraphQL configuration, `setSharedCookies()`, `defaultHeaders`, `JSONResponse` assertions, Zod schemas
- `node_modules/codeceptjs/docs/assertions.md` — built-in `see*` assertions, `ExpectHelper`, `codeceptjs/assertions` factories (use these instead of `if (cond) throw new Error(...)`)
- `node_modules/codeceptjs/docs/pageobjects.md` — porting Protractor page objects
- `node_modules/codeceptjs/docs/sessions.md`, `auth.md` — multi-user + login reuse
- `node_modules/codeceptjs/docs/effects.md` — `tryTo`, `retryTo`, `within`
- `writing-codeceptjs-tests` — per-spec rewrite playbook (drive via MCP, learn locators, commit verified steps)
- `debugging-codeceptjs-tests` — **use on every failing test from the first full run** (breakpoint, inspect live page via MCP, fix on the fly)
- `codeceptjs-auth` — replace UI re-login in every `beforeEach`
- `codeceptjs-fundamentals` — run **after** migration to confirm the new setup is wired correctly
