---
name: codeceptjs-fundamentals
description: Run first when working with any CodeceptJS 4 project. Compact primer on the internals you must know — configuration, the `I` actor and helpers, the DI container and `inject()`, custom helpers (and the rule that `I` is unreachable from inside one), plugins as hook listeners, the `await` rule, and where to list available actions. Then reads the project's config and reports which helper, plugins, env switching, and page objects are actually active. Other CodeceptJS skills depend on this output.
---

# CodeceptJS Fundamentals

Two jobs: teach the concepts you need to read CodeceptJS code without making things up, and report what *this* project has configured. Do both, in order.

---

## Concepts

### Module system
CodeceptJS 4 is **ESM-first**. Tests, configs, page objects, and helpers use `import`/`export`; the project's `package.json` has `"type": "module"`. CodeceptJS 3 was CommonJS-based — migrating from 3.x means converting `require()` / `module.exports` and updating `package.json`. **TypeScript** works either way: name the config `codecept.conf.ts`, add a loader entry like `require: ['tsx/cjs']` (or `ts-node/register`), and write tests as `.ts` files.

### Configuration
`codecept.conf.{js,ts,mjs,cjs}` at the repo root. Top-level keys: `helpers`, `plugins`, `include`, `ai`, `bootstrap`/`teardown`, `tests`, `output`, `timeout`. TypeScript configs declare a loader in `require: [...]` (`tsx/cjs`, `ts-node/register`, `ts-node/esm`). Multiple env-specific files (`codecept.ci.conf.js`, …) are selected via `--config <file>`. The `@codeceptjs/configure` package mutates the resolved config at load time (`setHeadlessWhen`, `setBrowser`, `setCommonPlugins`, `setWindowSize`) — static fields can lie until you grep for that import.

### `I` and helpers
`I` is the actor. Every `I.<method>(...)` is dispatched to whichever active helper provides that method. Built-in helpers contribute different surfaces: web (Playwright, Puppeteer, WebDriver — overlapping core actions plus helper-specific extras), API (REST, GraphQL), AI, mobile (Appium), utility (FileSystem). The active helpers are exactly the keys under `helpers` in config.

### `inject()` and the DI container
Everything testable lives in a global container — the actor, every helper, every page object listed in `include`, every custom step module, every support object. Inside a Scenario you destructure from the test signature: `Scenario('...', ({ I, loginPage }) => { ... })`. Inside a *file* (page object class, data factory, custom helper module) call `const { I } = inject()` once at the top to pull what you need from the container. The names available are exactly the keys in `include`.

### Custom helpers
Custom helpers extend `Helper` (from `codeceptjs`) and contribute new `I.<method>` calls. They register under `helpers` in config alongside built-ins. **You cannot call `I.*` from inside a custom helper — `I` does not exist in helper scope.** To compose with another helper, reach for it via `this.helpers['<Name>']` (e.g. `this.helpers['Playwright'].page` or `this.helpers['REST'].sendGetRequest(...)`). Helpers exist to expose new low-level capabilities; tests stay in the `I.*` vocabulary.

### Plugins and hooks
Plugins are event listeners. CodeceptJS emits lifecycle events on a global dispatcher; any plugin can subscribe. Events include `suite.before`/`after`, `test.before`/`started`/`passed`/`failed`/`after`, `step.before`/`started`/`passed`/`failed`/`after`, `hook.passed`/`failed`, `multiple.before`/`after`. Plugins react — taking screenshots, retrying, healing, writing artifacts, pausing. Built-ins live under `node_modules/codeceptjs/lib/plugin/`; the full event list is in `node_modules/codeceptjs/lib/event.js`. A plugin is registered under `plugins` with `enabled: true`. `setCommonPlugins()` from `@codeceptjs/configure` enables a recommended bundle silently.

### Plugins worth knowing
- `retryFailedStep` — re-runs a transient action failure
- `screenshotOnFail` — saves a screenshot for every failed step
- `pageInfo` — dumps URL, HTML, console output on failure
- `auth` — session reuse for login (see the `codeceptjs-auth` skill)
- `aiTrace` — per-step screenshots/HTML/ARIA/console for AI debugging

### Test file structure
One `Feature(...)` per file with one or more `Scenario(...)` blocks inside it. CodeceptJS does **not** allow nested suites or multiple Features in the same file. Hooks: `Before`, `After`, `BeforeSuite`, `AfterSuite`, plus `Fail((test, err) => { ... })` for failure-only cleanup. Page objects can expose `_before`, `_after`, `_afterSuite` lifecycle methods so per-page setup lives next to the page.

### Locators
Most actions accept a locator as a plain string (semantic — visible text, label, placeholder, `name`) or an object (`{ css }`, `{ xpath }`, `{ role, name }`, `{ id }`, `{ aria }`). Prefer ARIA `{ role, name }` for resilience to markup changes; semantic strings for prototyping; CSS / XPath as fallback. The `locate(...)` builder composes complex queries (`.withClass`, `.withText`, `.inside`, `.and`, `.andNot`). Almost every action method takes an optional context arg that narrows the search to a subtree: `I.click('Save', '.modal')`.

### Auto-waiting
Action methods (`click`, `fillField`, `selectOption`, …) automatically wait for the element to exist and become interactable before acting. Explicit `I.waitFor*` calls are needed only when the next condition isn't tied to an interaction — a modal appearing after a network call, a spinner disappearing, a value updating. Avoid `I.wait(N)` (raw seconds) unless nothing else fits.

### Assertions
CodeceptJS ships built-in browser assertions: `I.see`, `I.dontSee`, `I.seeElement`, `I.dontSeeElement`, `I.seeInCurrentUrl`, `I.seeInTitle`, `I.seeInField`, `I.seeNumberOfElements`, `I.seeCookie`, `I.seeCheckboxIsChecked`, etc. Use these instead of an external `expect()` library — they produce clear failure messages and integrate with the recorder. For non-DOM assertions, use `grab*` plus any assertion library: `const title = await I.grabTitle(); expect(title).toEqual('My App')`.

### `await` inside tests
CodeceptJS queues steps onto an internal recorder; the framework chains them, you do not. **Use `await` only when you need a return value** — `await I.grabTextFrom(...)`, `await I.grabCookie(...)`, or when calling a user-defined `async` function. Plain action steps (`I.click`, `I.fillField`, `I.see`) do not need `await`. Same rule inside `within(...)` and `pause()` callbacks. Sprinkling unnecessary `await` doesn't break anything, but signals you don't trust the recorder.

### `secret()` for sensitive values
Wrap passwords, tokens, API keys so they're masked in logs, step output, and trace artifacts: `I.fillField('Password', secret(process.env.PASSWORD))`. Imported from `codeceptjs`. Use anywhere a value would otherwise leak through verbose output, trace files, or AI prompts.

### Sessions and `within`
- `session(name, fn)` runs `fn` in a parallel browser context — for multi-user Scenarios (chat, multi-tenant). Combined with the `auth` plugin, each session can log in as a different role.
- `within(locator, fn)` scopes locator resolution inside `fn` to the subtree under `locator`. `within({ frame: '#editor' }, fn)` switches into an iframe for the callback. Both can return values (`await within(..., () => I.grabTextFrom(...))`).

### Parallel runs
`npx codeceptjs run-workers <N>` splits Scenarios across N Node worker threads; results aggregate in the main process. The config can also describe **profiles** (different browsers, viewports, environments) via the `multiple` block; launch with `npx codeceptjs run-multiple <profile>`.

### Listing available actions
Don't memorize — list them at runtime against the active config:
- `npx codeceptjs list` prints every available `I.<method>`.
- The CodeceptJS MCP server's `list_actions` tool returns the same data, grouped by helper, with signatures.

Use these before suggesting any method, especially in projects with custom helpers.

---

## Read this project

Open the active config (resolve via `package.json` scripts and CI workflows if multiple files exist). Extract: which helper(s) and any non-default behaviour (browser, strict, navigation, base URL, viewport, env-driven values); which plugins (incl. anything `setCommonPlugins()` injects); AI provider + the env var its key requires; how environments are selected (`--config` vs `process.env.*` branching, plus any `setHeadlessWhen`-style mutations); page object names from `include`; any custom helpers (entries pointing at local files).

## Report

Short prose summary covering the items above. Flag env-driven values explicitly — don't claim a fixed value when it's `process.env.BROWSER || 'chromium'`. Flag conflicts (static `show: true` overridden by `setHeadlessWhen(CI)`; `auth` configured but the credential env vars are missing from the current shell or `.env.example`). If no config exists at the repo root and no `--config` is referenced anywhere, recommend `npx codeceptjs init .` and stop.

## Pointers

- `node_modules/codeceptjs/docs/configuration.md` — config reference
- `node_modules/codeceptjs/docs/typescript.md` — TS loader options
- `node_modules/codeceptjs/docs/helpers.md` — helper concepts and method catalogs
- `node_modules/codeceptjs/docs/custom-helpers.md` — writing your own
- `node_modules/codeceptjs/docs/plugins.md` — plugin authoring + built-ins
- `node_modules/codeceptjs/docs/hooks.md` — suite/test/step hook semantics
- `node_modules/codeceptjs/lib/event.js` — every event the dispatcher emits
- `@codeceptjs/configure` (npm) — the mutator API surface
