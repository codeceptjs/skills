---
name: codeceptjs-fundamentals
description: "Run first when working with any CodeceptJS 4 project. Compact primer on the internals you must know — configuration, the `I` actor and helpers, the DI container and `inject()`, custom helpers (and the rule that `I` is unreachable from inside one), plugins as hook listeners, and the `await` rule. Then runs a four-step discovery against this project: `codeceptjs check` to verify the setup loads, read the config, run `codeceptjs list` to enumerate available `I.*` actions, run `codeceptjs dry-run` to enumerate existing tests — and reports which helper, plugins, env switching, page objects, custom actions, and test suites are actually active. Other CodeceptJS skills depend on this output."
---

# CodeceptJS Fundamentals

Two jobs: teach the concepts you need to read CodeceptJS code without making things up, and report what *this* project has configured. Do both, in order.

---

## Concepts

### Module system
CodeceptJS 4 is **ESM and TypeScript only**. Tests, configs, page objects, and helpers use `import`/`export`; `package.json` **must** have `"type": "module"` — if it isn't there yet, add it before doing anything else (without it, every `.js` file is parsed as CommonJS and imports fail). **TypeScript** is first-class: name the config `codecept.conf.ts`, add a loader entry like `require: ['tsx/cjs']` (or `ts-node/register`), and write tests as `.ts` files.

If the project is on **CodeceptJS 3.x or still uses CommonJS** (`require()` / `module.exports`, no `"type": "module"`, removed helpers/plugins like `autoLogin` or `Nightmare`), stop here and run the **`migrate-codeceptjs-4`** skill — it walks the full upgrade path (Node bump, ESM conversion, helper/plugin replacements, AI/Zod/effects API changes, `noGlobals`, dependency bumps, verify). Don't try to half-fix individual files; the migration is a whole-project change.

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
- `screenshot` — saves a screenshot when a step matches the trigger (default `on: 'fail'`); set `slides: true` to also produce a `output/records.html` slideshow (replaces the old `stepByStepReport`)
- `pageInfo` — dumps URL, HTML, console output on failure
- `auth` — session reuse for login (see the `codeceptjs-auth` skill)
- `aiTrace` — per-step screenshots/HTML/ARIA/console for AI debugging (default `on: 'step'`; set `on: 'fail'` to capture only failures)
- `pause` — interactive pause (replaces `pauseOn` / `pauseOnFail`; default `on: 'fail'`)
- `heal` — AI-suggested fixes for broken action steps (disabled in `--debug` mode)
- `screencast` — records a video / animated frames of the run (replaces `subtitles`)
- `browser` — CLI-only override of browser helper config; see the next section

### Running plugins from the CLI
Plugins are normally enabled in `codecept.conf.{js,ts}`, but any plugin can be turned on or reconfigured for a single run via `-p <plugin>` on the runner. Args chain with `:` (or `;` inside one arg):

```bash
npx codeceptjs run -p aiTrace                      # enable aiTrace for this run
npx codeceptjs run -p screenshot:on=step           # screenshot every step
npx codeceptjs run -p pause:on=file:path=tests/login_test.js;line=43
npx codeceptjs run -p browser:hide:browser=firefox:windowSize=1280x800
```

`screenshot`, `pause`, `aiTrace`, and `heal` share a unified **`on=` parameter** that picks when they fire:

| `on=` value | Fires when | Extra args |
|---|---|---|
| `fail` | a step fails (default for screenshot / pause / heal) | — |
| `step` | every step (default for aiTrace) | — |
| `test` | after each test | — |
| `file` | execution reaches a file/line | `path=<file>[;line=<N>]` |
| `url` | browser URL matches a pattern | `pattern=<glob>` (`*` wildcards) |

The **`browser` plugin** is CLI-only and overrides the active browser helper without touching the config file — useful for one-off env variants and CI matrix legs:

```bash
npx codeceptjs run -p browser:show                              # force visible
npx codeceptjs run -p browser:hide                              # force headless
npx codeceptjs run -p browser:browser=firefox                   # switch browser
npx codeceptjs run -p browser:windowSize=1024x768
npx codeceptjs run -p browser:hide:browser=webkit:windowSize=800x600
```

Requires `@codeceptjs/configure` installed. It translates `browser=` per helper (Puppeteer's `product`, Playwright's `browser`) and injects `--headless` into WebDriver capability args when toggling `hide`.

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

---

## Discover this project

Four steps, in order. Don't skip — guesses about helpers, custom actions, or what tests exist will be wrong as often as they're right.

### 1. Verify the setup loads

```sh
npx codeceptjs check -c <config>          # validates config, container, helpers, plugins, page objects, hooks, tests, defs
```

Each item prints a pass/fail line, so the output doubles as a quick inventory of what the project has wired up. If anything fails here, fix it before running `list` or `dry-run` — a broken helper or unresolved page object will distort their output. Skipping this step also means you won't notice a missing dependency, an `auth` plugin pointed at a non-existent login route, or a custom helper that throws at construction.

### 2. Read the active config

Open `codecept.conf.{js,ts,mjs,cjs}` (resolve via `package.json` scripts and CI workflows if multiple files exist — note the path; you'll pass it to `-c` in steps 3 and 4). Extract: which helper(s) and any non-default behaviour (browser, strict, navigation, base URL, viewport, env-driven values); which plugins (incl. anything `setCommonPlugins()` injects); AI provider + the env var its key requires; how environments are selected (`--config` vs `process.env.*` branching, plus any `setHeadlessWhen`-style mutations); page object names from `include`; any custom helpers (entries pointing at local files).

### 3. List available actions

```sh
npx codeceptjs list -c <config>          # every I.<method>, grouped by helper, with signature
npx codeceptjs list --docs -c <config>   # adds JSDoc + docs/webapi/* prose under each action
npx codeceptjs list --action <name> -c <config>   # single action; I. prefix optional; implies --docs
```

Run `list` against the discovered config before suggesting any method — especially in projects with custom helpers, where the available `I.*` surface differs from the built-in catalog. The CodeceptJS MCP server's `list_actions` tool returns the same data programmatically.

### 4. List existing tests

```sh
npx codeceptjs dry-run -c <config>            # suite + test names that the config would load
npx codeceptjs dry-run --steps -c <config>    # also prints queued I.* steps inside each test
npx codeceptjs dry-run --grep "@smoke" -c <config>   # filter by name; --features / --tests narrow file kind
npx codeceptjs dry-run --debug --grep '<test>' --numbers --no-ansi -c <config>   # numbered steps, no ANSI
```

`dry-run` walks the test files the active config picks up and prints them without executing — confirming both **which tests exist** and (with `--steps`) **what each Scenario would do** before any browser spins up.

`--numbers` (paired with `--debug`, `--steps`, or `--verbose`) prefixes each leaf step with a per-test 1-based index. The numbering matches the `pauseAt: N` parameter on the MCP `run_test` tool — so this is the canonical way to discover step indices for programmatic breakpoints. `--no-ansi` strips colors / ANSI escapes so the output is clean for LLM consumption or piping to other tools.

For Gherkin step definitions specifically, `npx codeceptjs gherkin:steps -c <config>` lists registered step patterns.

## Report

Short prose summary covering the items above. Flag env-driven values explicitly — don't claim a fixed value when it's `process.env.BROWSER || 'chromium'`. Flag conflicts (static `show: true` overridden by `setHeadlessWhen(CI)`; `auth` configured but the credential env vars are missing from the current shell or `.env.example`). If no config exists at the repo root and no `--config` is referenced anywhere, recommend `npx codeceptjs init .` and stop. **If the project is on CodeceptJS 3.x or CommonJS, recommend the `migrate-codeceptjs-4` skill and stop** — discovery output for a pre-4 project will misrepresent the available APIs.

## Pointers

- `node_modules/codeceptjs/docs/configuration.md` — config reference
- `node_modules/codeceptjs/docs/typescript.md` — TS loader options
- `node_modules/codeceptjs/docs/helpers.md` — helper concepts and method catalogs
- `node_modules/codeceptjs/docs/custom-helpers.md` — writing your own
- `node_modules/codeceptjs/docs/plugins.md` — plugin authoring + built-ins
- `node_modules/codeceptjs/docs/hooks.md` — suite/test/step hook semantics
- `node_modules/codeceptjs/lib/event.js` — every event the dispatcher emits
- `@codeceptjs/configure` (npm) — the mutator API surface
