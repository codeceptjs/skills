---
name: migrate-testcafe-to-codeceptjs
description: "Port a TestCafe test suite to CodeceptJS 4. Trigger when the project contains `.testcaferc.{json,js,ts,cjs}`, `testcafe` in `devDependencies`, test files that import from `testcafe` (`Selector`, `ClientFunction`, `Role`, `RequestMock`, `RequestHook`, `RequestLogger`, `t` from a test signature), top-level `fixture('X').page(...)` + `test('y', async t => { ... })` blocks, `Selector(...)` chains (`.withText`, `.withAttribute`, `.nth`, `.find`, `.filter`, `.parent`, `.child`, `.sibling`), `await t.click(...)` / `t.typeText(...)` patterns, `t.useRole(...)`, `t.addRequestHooks(...)`, `t.eval(...)`, `ClientFunction(...)`, TestCafe Studio recordings, or a `tests/` directory matching `*.test.{js,ts}` whose contents start with `fixture(...)`. Walks the port end-to-end: inventory shared logic (TestCafe page-object-style modules, `Role` definitions, `ClientFunction` factories, `RequestMock` factories, custom Test Controller actions, fixture hooks), install CodeceptJS with the Playwright helper alongside TestCafe, port the config, split shared helpers into `WebExtra` (browser-driven via Playwright `page` — `ClientFunction` / `t.eval` ports here as `page.evaluate`) and `ApiExtras` (HTTP via REST helper, never `browserContext.request.*`), port TestCafe page objects to CodeceptJS page objects without inventing assertion/one-liner wrappers, replace `Selector(...)` chains with semantic strings / ARIA / `locate()` / `{ css }`, **strip excess `await` from every action call** (CodeceptJS auto-queues — `await` is only for grabs), convert specs (handing off to `writing-codeceptjs-tests`), replace `Role` + `t.useRole` with the `auth` plugin, swap `RequestMock` for `I.mockRoute`, fold `t.expect(sel.X).Y(...)` chains into `I.see*` / `ExpectHelper` / `codeceptjs/assertions`, then decommission TestCafe."
---

# Migrate TestCafe → CodeceptJS 4

TestCafe and CodeceptJS share a lot at the surface — both expose a single test-controller verb-set (`t.*` / `I.*`), both have lazy chainable selectors, both ship with role-based auth and screenshot/video support. The migration is mostly mechanical, but three foundational differences drive the work:

1. **CodeceptJS does not need `await` on actions.** The recorder auto-queues every `I.*` call. TestCafe forces `await` on every action (`await t.click(...)`); CodeceptJS forbids it on actions and reserves it for grabs (`await I.grabTextFrom(...)`). **This is the single biggest mechanical edit during spec conversion** — strip every `await` from before `I.click`, `I.fillField`, `I.see*`, `I.waitFor*`, and page-object method calls that return void.
2. **Helpers, not a bundled proxy.** TestCafe runs as an HTTP/HTTPS proxy that injects automation into pages; CodeceptJS dispatches `I.*` to a configured helper. **Playwright recommended** — closest feel, fastest, supports all three engines (Chromium / Firefox / WebKit) the same way TestCafe did.
3. **First-class abstractions.** Page objects, multi-user `session(...)`, the `auth` plugin, custom helpers, and the `customLocator` plugin are built in. TestCafe projects accumulate ad-hoc versions of these (Selector-property classes, `Role` factories, `ClientFunction` factories) — the migration consolidates them onto framework idioms.

Authoritative references: `node_modules/codeceptjs/docs/basics.md`, `locators.md`, `playwright.md`, `custom-helpers.md`, `pageobjects.md`.

## When to trigger

Any of:

- `.testcaferc.{json,js,ts,cjs}` at the repo root.
- `testcafe` listed in `devDependencies`.
- Imports from `testcafe` (`Selector`, `ClientFunction`, `Role`, `RequestMock`, `RequestHook`, `RequestLogger`).
- Test files with top-level `fixture('X').page(...)` + `test('y', async t => { ... })`.
- Code uses `Selector(...).withText(...)` / `.withAttribute(...)` / `.nth(...)` / `.find(...)` / `.filter(...)` chains, `t.useRole(...)`, `t.addRequestHooks(...)`, `t.eval(...)`, or `ClientFunction(...)`.
- A `tests/` directory of `*.test.{js,ts}` whose contents start with `fixture(...)`.
- The user says "migrate / port / convert from TestCafe".

## What does not migrate

Be honest up-front:

- **Proxy-based architecture** — TestCafe runs as a man-in-the-middle proxy and rewrites pages to inject its driver. CodeceptJS uses Playwright (CDP) or WebDriver. The trade-off: lose driverless setup, gain Playwright's speed and ergonomics. Rare TLS / CORS tricks that relied on the proxy will need rethinking.
- **TestCafe Studio recordings** — UI-recorded tests must be re-authored. Use the `writing-codeceptjs-tests` MCP scaffold-and-pause mode to recreate them against the live browser.
- **TestCafe `RequestHook` / `RequestLogger`** — replaced piecewise: hooks → `I.mockRoute()`; loggers → `page.on('request' | 'response')` inside `WebExtra` if you really need a transcript, or anchor on UI outcomes instead.
- **`disablePageCaching`, `quarantineMode` finer tuning** — Playwright handles caching per context; quarantine maps roughly to `retry: N` but lacks the same heuristics.
- **Mobile testing via `testcafe-browser-provider-*` packages** — use Playwright's mobile emulation (`devices['iPhone 13']`) or the `Appium` helper for real devices.
- **TestCafe Cloud / Dashboard** — replaced by `@testomatio/reporter` or another CodeceptJS-compatible reporter.

## Workflow

Run phases in order. Commit at each boundary so any regression is bisectable.

### 1. Inventory the TestCafe project

Before touching anything, build a picture. Two passes.

**Shape of the project** — grep / `wc -l` for cost predictors:

- `.testcaferc.{json,js,ts,cjs}` — which keys are in use (`browsers`, `src`, `concurrency`, `selectorTimeout`, `assertionTimeout`, `pageLoadTimeout`, `screenshots`, `videoPath`, `clientScripts`, `quarantineMode`, `stopOnFirstFail`, `reporter`)
- test file count + glob (TestCafe has no required suffix; commonly `*.test.{js,ts}` or anything under `tests/`)
- count occurrences of `Selector(`, `ClientFunction(`, `Role(`, `RequestMock(`, `t.useRole(`, `t.eval(`, `t.addRequestHooks(`, `.withText(`, `.withAttribute(`, `.nth(`, `.find(`, `.filter(`, `.parent(`, `.child(`, `.sibling(` — each maps to a known replacement pattern

**Shared logic** — TestCafe projects accumulate four kinds of shared abstractions even without framework support:

- **Page-object-style modules** — classes whose properties are `Selector(...)` references and whose methods drive `t.*`. Usually under `tests/page-objects/`, `tests/pages/`, or `<feature>.po.{js,ts}`. Port directly to CodeceptJS page objects.
- **`Role` definitions** — every `const admin = Role('https://x/login', async t => { ... })`. These are TestCafe's session-cached login flows; their replacement is the **`auth` plugin** (phase 8).
- **`ClientFunction` factories** — `const getURL = ClientFunction(() => window.location.href)`. Each becomes a method on `WebExtra` using `page.evaluate`.
- **`RequestMock` factories / hook files** — `RequestMock().onRequestTo('/api/x').respond(...)`. Each becomes an `I.mockRoute(...)` call, either inline in tests or wrapped on `WebExtra` if reused widely.
- **Fixture hooks** — `fixture(...).beforeEach(...)` / `.before(...)` / `.after(...)`. Become CodeceptJS `Before` / `BeforeSuite` hooks in the corresponding test file.
- **Custom Test Controller methods** — projects sometimes extend `t` via mixins; treat them as helper methods and split UI vs HTTP into `WebExtra` / `ApiExtras`.

Produce a short inventory: every shared abstraction with its current location and planned CodeceptJS destination. The user reviews before any code is written.

### 2. Install CodeceptJS alongside TestCafe

`npx codeceptjs init` and pick the **Playwright** helper. Playwright covers the same three engines TestCafe supported (Chromium / Firefox / WebKit) with one config. Do not remove TestCafe yet — both run in parallel through the migration.

### 3. Port the config

Map `.testcaferc.{json,js}` keys → `codecept.conf.{js,ts}`:

| TestCafe | CodeceptJS 4 (`Playwright` helper) |
|---|---|
| `browsers: ['chrome']` / `['firefox']` / `['safari']` | `helpers.Playwright.browser: 'chromium'` / `'firefox'` / `'webkit'` |
| `browsers: ['chrome:headless']` | `helpers.Playwright.show: false` (or rely on `setHeadlessWhen(CI)`) |
| `src: ['tests/**/*.test.js']` | `tests: './tests/**/*_test.{js,ts}'` |
| fixture `.page('https://x')` | `helpers.Playwright.url: 'https://x'` |
| `selectorTimeout` / `assertionTimeout` | `helpers.Playwright.waitForTimeout` |
| `pageLoadTimeout` | `helpers.Playwright.timeout` |
| `concurrency: N` | CLI: `npx codeceptjs run-workers N` |
| `screenshots.path` / `videoPath` | top-level `output: './output'` |
| `screenshots.takeOnFails: true` | plugin `screenshot` with `on: 'fail'` |
| `videoPath` set | `helpers.Playwright.video: true` |
| `clientScripts: ['inject.js']` | `WebExtra` method using `page.addInitScript`, or `bootstrap()` |
| `quarantineMode` | top-level `retry: N` |
| `stopOnFirstFail: true` | CLI: `--bail` |
| `reporter: 'spec'` | drop (Mocha default) or plugin |
| `hostname` / `port` (proxy) | drop — Playwright manages |

### 4. Port shared abstractions

This is the bedrock. Do it before any spec rewrite — every spec rewrite shrinks because the verbs it needs (`I.doSmth(...)`) already exist.

**Hard rule for shared helper code.** Every reusable browser / HTTP function becomes a method on a custom CodeceptJS helper. **Split across two helpers by the kind of operation** — they have different access patterns and different correct APIs:

- **`WebExtra`** (`lib/helpers/WebExtra.js`) for **browser-driven** operations — anything that needs the open page, DOM, init scripts, storage, network-response waits. **This is where every `ClientFunction` and `t.eval` body lands, as `page.evaluate(...)`.** Reaches `this.helpers['Playwright'].page` / `.browserContext`.
- **`ApiExtras`** (`lib/helpers/ApiExtras.js`) for **pure HTTP** operations — programmatic login, seed/teardown data, CRUD against an API. Reaches `this.helpers['REST']` (or `GraphQL`). See `node_modules/codeceptjs/docs/api.md` for REST helper configuration.

Register both helpers under `helpers` in `codecept.conf.{js,ts}`.

**Never call `this.helpers['Playwright'].browserContext.request.*` for API work.** That bypasses the REST + `JSONResponse` stack — no step logging, no `I.seeResponseCodeIsSuccessful` assertions, no shared headers. If the API needs the same auth as the browser, share cookies once at the top of the config:

```js
import { setSharedCookies } from '@codeceptjs/configure'
setSharedCookies()
```

…or set `defaultHeaders` on the REST helper for token-based auth, or use `I.amBearerAuthenticated(secret(token))` per test. All three patterns are covered in `api.md`.

**WebExtra example** — `ClientFunction` ports here:

```js
import Helper from '@codeceptjs/helper'

export default class WebExtra extends Helper {
  async grabLocationHref() {
    const { page } = this.helpers['Playwright']
    return page.evaluate(() => window.location.href)
  }

  async setLocalStorage(key, value) {
    const { page } = this.helpers['Playwright']
    await page.evaluate(([k, v]) => localStorage.setItem(k, v), [key, value])
  }

  async injectClientScript(path) {
    const { page } = this.helpers['Playwright']
    await page.addInitScript({ path })
  }
}
```

**ApiExtras example** — `RequestMock` for *real* HTTP calls (seeding test data, not mocking responses) goes here:

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

For **request mocking** (`RequestMock().onRequestTo(...).respond(...)`), see phase 10 — that uses `I.mockRoute` (Playwright), not `ApiExtras`.

**Helper code style** — applies to both:

- All `import` statements at the **top of the file**. Never `const fs = await import('node:fs/promises')` inside a method.
- Use built-in assertions (`I.seeResponseCodeIsSuccessful` for API, `I.seeElement` for browser), `ExpectHelper`, or factories from `codeceptjs/assertions` — **never** `if (cond) throw new Error('...')`. Failures must render as proper assertion errors. See `node_modules/codeceptjs/docs/assertions.md`.

**Other destinations** from the phase 1 inventory:

- **TestCafe page-object-style module** → CodeceptJS **page object class** under `pages/`. **Port conservatively** — keep only the methods the original module had; do not invent new wrappers during migration. `Selector(...)` properties (`this.usernameField = Selector('#user')`) become locator-string fields (`fields = { usernameField: '#user' }`); methods rewrite with `const { I } = inject()` at the top, calling `I.fillField`, `I.click`, and any `I.*` verb the `WebExtra` / `ApiExtras` helpers now contribute. Strip the `async t =>` plumbing — methods receive their args directly. Register under `include` in `codecept.conf.{js,ts}` so the page object auto-injects into Scenarios.

  Page-object anti-patterns to avoid (unless the original TestCafe module already had them):
  - **Assertion methods** (`checkTitle() { I.seeElement(...) }`) — page objects are action verbs (`fillForm`, `submitOrder`); let assertions live in the test.
  - **One-liner wrappers** around a single `I.click` / `I.see*` / `I.grabTextFrom` — the wrapper buys nothing over calling `I.*` from the test.
  - **Methods used by only one test** — leave the steps in the test.
  - **`if (cond) throw new Error(...)`** in any method — use `I.see*`, `I.seeNumberOfElements`, `ExpectHelper`, or `codeceptjs/assertions` factories instead.

- **Shared `Selector` constants** → fields on the relevant page object. No free-floating `selectors.{js,ts}`.
- **`Role` definitions** → `auth` plugin role definitions (phase 8). If the `Role` body called the UI to log in, port it as a `WebExtra` method first; if it hit the API, port it as `ApiExtras`. The `auth` plugin then calls that method.
- **Pure utility modules** that don't touch the browser → plain ES modules, imported where needed.

Sanity-check before moving on: `npx codeceptjs check -c <config>` must pass, and `npx codeceptjs list -c <config>` must show every ported helper method as an `I.*` action contributed by `WebExtra` or `ApiExtras`.

### 5. Convert spec files

One file at a time, leaning on the abstractions from phase 4. Hand off the per-spec work to the **`writing-codeceptjs-tests`** skill — it drives the live browser via MCP and verifies each step before committing.

| TestCafe | CodeceptJS 4 |
|---|---|
| File `*.test.{js,ts}` | `*_test.{js,ts}` |
| `fixture('X').page('/x')` | `Feature('X')` at top + `Before(({ I }) => I.amOnPage('/x'))` |
| `test('y', async t => { ... })` | `Scenario('y', ({ I }) => { ... })` — drop `async t =>`, take `({ I, … })` from the test signature |
| `.beforeEach(async t => { ... })` | `Before(({ I }) => { ... })` |
| `.afterEach(async t => { ... })` | `After(({ I }) => { ... })` |
| `.before(...)` / `.after(...)` | `BeforeSuite(...)` / `AfterSuite(...)` |
| `t.navigateTo('/x')` | `I.amOnPage('/x')` |
| `await loginPage.login(u, p)` | `loginPage.login(u, p)` — no `await` on void page-object methods |

**Strip excess `await`** — this is the single biggest mechanical edit. TestCafe required `await` on every action; CodeceptJS forbids it on actions and reserves it for grabs. Convention:

```js
I.click('Save')                        // no await
I.fillField('Email', 'u@t.com')        // no await
I.see('Saved')                         // no await
I.waitForElement('.toast', 3)          // no await
const text = await I.grabTextFrom('h1')   // await — grabs return data
const ok = await tryTo(() => I.click('Accept'))  // await — effects can return values
```

If a step in the original used `t.ctx.foo = ...` to thread state through one test, store it in a plain `let` declared in the Scenario callback. `t.fixtureCtx` (suite-wide state) becomes a module-level variable, or a `BeforeSuite`-populated object.

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

CodeceptJS priority — pick the highest that fits. TestCafe's lazy chainable `Selector` lines up well with CodeceptJS's `locate()` builder, but most chains shrink considerably because semantic strings cover what `.withText` was doing.

1. **Semantic strings** — button text, label, placeholder, link text: `I.click('Save')`, `I.fillField('Email', 'u@t.com')`. Covers `Selector('button').withText('Save')` cleanly.
2. **ARIA roles** — `I.click({ role: 'button', name: 'Sign In' })`. Strong default for modern apps.
3. **`locate()` builder** — `I.click(locate('.row').withText('Acme').inside('table'))`. Direct equivalent of TestCafe `Selector('.row').withText('Acme').find('.btn')` style chains.
4. **CSS / XPath / attribute objects** — `{ id: 'foo' }`, `{ name: 'email' }`, `{ css: '[data-test=submit]' }`, `{ xpath: '//div[@id="x"]' }`. Fallback.

| TestCafe Selector chain | CodeceptJS 4 |
|---|---|
| `Selector('.btn')` | `'.btn'` |
| `Selector('button').withText('Submit')` | `'Submit'` (semantic) or `locate('button').withText('Submit')` |
| `Selector('button').withExactText('Submit')` | `locate('button').withTextEquals('Submit')` — or assert via `I.seeTextEquals('Submit', 'button')` |
| `Selector('input').withAttribute('name', 'email')` | `{ name: 'email' }` |
| `Selector('input').withAttribute('data-test', /^submit/)` | `{ css: 'input[data-test^="submit"]' }` |
| `Selector('.row').nth(0)` | `step.opts({ elementIndex: 1 })` |
| `Selector('.row').nth(-1)` | `step.opts({ elementIndex: 'last' })` |
| `Selector('.row').find('.btn')` | `locate('.btn').inside('.row')` — or context arg `I.click('.btn', '.row')` |
| `Selector('.parent').child('.kid')` | `locate('.kid').inside('.parent')` |
| `Selector('.row').filter('.active')` | `locate('.row').withClass('active')` |
| `Selector('button').parent('.toolbar')` | n/a one-liner; restructure as `I.click('button', '.toolbar')` |
| `Selector(t => t.foo)` (function selectors) | method on `WebExtra` using `page.locator` / `page.evaluate` |
| custom `t.fixtureCtx.selector = Selector(...)` | page-object field |

`step.opts(...)` comes from `import step from 'codeceptjs/steps'`.

### 7. Actions, assertions, grabs

TestCafe's Test Controller (`t`) and CodeceptJS's actor (`I`) line up closely — most actions are a verb rename. The big edits are dropping `await` from actions, collapsing `t.expect(sel.X).Y(...)` chains into single `I.see*` calls, and rewriting `t.eval` / `ClientFunction` into helper methods.

| TestCafe | CodeceptJS 4 |
|---|---|
| `await t.click(sel)` | `I.click(sel)` |
| `await t.typeText(sel, 'x')` | `I.fillField(sel, 'x')` (clears by default — same as TestCafe with `replace: true`) |
| `await t.typeText(sel, 'x', { replace: false })` | `I.appendField(sel, 'x')` |
| `await t.pressKey('enter')` | `I.pressKey('Enter')` |
| `await t.hover(sel)` | `I.moveCursorTo(sel)` |
| `await t.dragToElement(sel, target)` | `I.dragAndDrop(sel, target)` |
| `await t.takeScreenshot('x.png')` | `I.saveScreenshot('x.png')` |
| `await t.takeElementScreenshot(sel, 'x.png')` | `I.saveElementScreenshot(sel, 'x.png')` |
| `await t.resizeWindow(W, H)` | `I.resizeWindow(W, H)` |
| `await t.maximizeWindow()` | `I.resizeWindow('maximize')` |
| `await t.setNativeDialogHandler(fn)` | `I.acceptPopup()` / `I.cancelPopup()` per dialog |
| `await t.switchToIframe(sel)` | `within({ frame: sel }, () => { ... })` |
| `await t.switchToMainWindow()` | (end of `within` block) |
| `await t.openWindow(url)` | `session('w2', () => I.amOnPage(url))` |
| `await t.eval(() => document.title)` | `await I.executeScript(() => document.title)` — or method on `WebExtra` |
| `ClientFunction(() => window.location.href)()` | `await I.grabCurrentUrl()` (or `webExtra.grabLocationHref()` from phase 4) |
| `await t.wait(N)` (N ms) | `I.wait(N / 1000)` — CodeceptJS uses **seconds**; avoid in committed tests |
| `await t.getBrowserConsoleMessages()` | `await I.grabBrowserLogs()` |
| `await t.expect(sel.innerText).eql('X')` | `I.seeTextEquals('X', sel)` — or `I.see('X', sel)` for "contains" |
| `await t.expect(sel.innerText).contains('X')` | `I.see('X', sel)` |
| `await t.expect(sel.value).eql('X')` | `I.seeInField(sel, 'X')` |
| `await t.expect(sel.checked).ok()` | `I.seeCheckboxIsChecked(sel)` |
| `await t.expect(sel.classNames).contains('active')` | `I.seeElementHasClass(sel, 'active')` |
| `await t.expect(sel.exists).ok()` | `I.seeElementInDOM(sel)` |
| `await t.expect(sel.exists).notOk()` | `I.dontSeeElementInDOM(sel)` |
| `await t.expect(sel.visible).ok()` | `I.seeElement(sel)` |
| `await t.expect(sel.visible).notOk()` | `I.dontSeeElement(sel)` |
| `await t.expect(sel.count).eql(N)` | `I.seeNumberOfElements(sel, N)` |
| `await t.expect(value).eql(expected)` | `const v = await I.grabXxxFrom(...); I.expectEqual(v, expected)` (ExpectHelper) |
| `await sel.innerText` (grab) | `await I.grabTextFrom(sel)` |
| `await sel.getAttribute('data-id')` | `await I.grabAttributeFrom(sel, 'data-id')` |
| `await sel.count` (grab) | `await I.grabNumberOfVisibleElements(sel)` |

`await` only on grabs. Plain actions queue automatically.

### 8. Sessions and auth

`Role(url, async t => { ... })` + `t.useRole(role)` → the **`auth` plugin**. Hand off to **`codeceptjs-auth`** for the setup walk-through. The plugin caches the post-login cookie/storage state and replays it per test, which is exactly what `Role` does in TestCafe. If phase 4 already ported the `Role` body into `ApiExtras` as `I.loginViaApi(...)` or into `WebExtra` as `I.login(...)`, the `auth` plugin's role definition just calls it.

For multi-user scenarios (TestCafe handled this via multiple roles + `t.useRole` swaps), use `session(...)` from `codeceptjs/effects`.

### 9. Fixtures, requests, tasks

| TestCafe | CodeceptJS 4 |
|---|---|
| `import users from './fixtures/users.json'` | `import users from './fixtures/users.json' with { type: 'json' }` |
| `t.request(...)` (TestCafe 1.20+) | `await I.sendPostRequest(...)` via the **REST helper**; wrap reusable flows in the `ApiExtras` helper from phase 4 |
| `clientScripts` (inject JS per page) | `helpers.Playwright.bootstrap` (per-context init script) or `WebExtra` method using `page.addInitScript` |
| Test data via `fixture('X').meta(...)` | `Scenario(..., { tag: '@x' })` plus a constants module |

REST helper auth: `setSharedCookies()` from `@codeceptjs/configure` shares the browser session with REST so the same user is logged in on both sides; alternatively set `defaultHeaders` for static tokens or `I.amBearerAuthenticated(secret(token))` per test. See `node_modules/codeceptjs/docs/api.md`.

### 10. Network mocking

`RequestMock().onRequestTo(url).respond(body, status, headers)` → `I.mockRoute(url, route => route.fulfill({ status, headers, body }))` (Playwright). `RequestHook` subclasses become route handlers too. Disable with `I.stopMockingRoute(url)`. For request *logging* (`RequestLogger`), there is no direct equivalent — anchor assertions on UI outcomes (`I.waitForText`, `I.see`) or, if you really need a request transcript for the test, attach `page.on('request' | 'response')` inside a `WebExtra` method. See `node_modules/codeceptjs/docs/playwright.md` § Mocking Network Requests.

### 11. Decommission TestCafe

Only after every spec is ported and CI is green: delete `.testcaferc.{json,js,ts,cjs}`, drop `testcafe` from `devDependencies` (plus any `testcafe-browser-provider-*`, `testcafe-reporter-*`, `testcafe-react-selectors`, `testcafe-vue-selectors` add-ons), remove the TestCafe CI jobs, uninstall any standalone TestCafe binary.

## Verify

1. `npx codeceptjs check -c <config>` — config + helper + plugin sanity.
2. `npx codeceptjs list -c <config>` — every ported helper method appears as an `I.*` action from `WebExtra` or `ApiExtras`; every page object's methods appear.
3. `npx codeceptjs dry-run --steps -c <config>` — every Scenario loads.
4. Full run: `npx codeceptjs run --steps -c <config>`. Failures are expected on first runs — drive each to a fix via the **`debugging-codeceptjs-tests`** skill (not `retry`, not blind rewrites). The migration is complete only when the whole converted suite is green.
5. Hand off to **`codeceptjs-run-analysis`** to inspect `output/trace_*/` artifacts (requires the `aiTrace` plugin enabled).
6. `grep -rE "\\bfixture\\(|\\btest\\(|Selector\\(|ClientFunction\\(|\\bRole\\(|t\\.click\\(|t\\.typeText\\(" tests/` — empty before deleting the original TestCafe directory.

## Pointers

- `node_modules/codeceptjs/docs/basics.md` — `I.*` vocabulary, locators, assertions, the `await` rule (the rule TestCafe users have to *un*-learn)
- `node_modules/codeceptjs/docs/playwright.md` — recommended helper; `mockRoute` for `RequestMock`; `evaluate` for `ClientFunction` / `t.eval` ports
- `node_modules/codeceptjs/docs/locators.md` — semantic / ARIA / `locate()` builder (replaces TestCafe Selector chains)
- `node_modules/codeceptjs/docs/custom-helpers.md` — `WebExtra` / `ApiExtras` patterns (extending `Helper`, reaching `this.helpers['Playwright']` / `this.helpers['REST']`)
- `node_modules/codeceptjs/docs/api.md` — REST / GraphQL configuration, `setSharedCookies()`, `defaultHeaders`, `JSONResponse` assertions, Zod schemas
- `node_modules/codeceptjs/docs/assertions.md` — built-in `see*` assertions, `ExpectHelper`, `codeceptjs/assertions` factories (use these instead of `if (cond) throw new Error(...)`)
- `node_modules/codeceptjs/docs/pageobjects.md` — porting TestCafe Selector-based page objects
- `node_modules/codeceptjs/docs/sessions.md`, `auth.md` — replaces `Role` + `t.useRole`
- `node_modules/codeceptjs/docs/effects.md` — `tryTo`, `retryTo`, `within` (replaces `t.switchToIframe`)
- `writing-codeceptjs-tests` — per-spec rewrite playbook (drive via MCP, learn locators, commit verified steps)
- `debugging-codeceptjs-tests` — **use on every failing test from the first full run** (breakpoint, inspect live page via MCP, fix on the fly)
- `codeceptjs-auth` — replace `Role` + `t.useRole`
- `codeceptjs-fundamentals` — run **after** migration to confirm the new setup is wired correctly
