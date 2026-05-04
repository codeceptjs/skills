---
name: writing-codeceptjs-tests
description: Use when writing a new CodeceptJS 4 test, extending an existing Scenario, or porting a manual test plan to code. Builds tests live — opens the real page through the CodeceptJS MCP server, queries ARIA/HTML to learn locators, runs each step incrementally to verify it works, then commits the verified sequence to a test file. Two authoring modes — Mode A (incremental `run_code`) for known flows, Mode B (scaffold-and-pause — write `I.amOnPage(...); pause();`, run via MCP, drive the live browser, replace `pause()` with the verified sequence) for greenfield / unknown flows. Never invents locators or flows from imagination; drives the actual browser. Trigger on any request to create, write, add, draft, or scaffold a CodeceptJS test, login flow, end-to-end check, or "test from scratch".
---

# Writing CodeceptJS 4 Tests

A test that was never executed during authoring is unreliable by definition. The right way to write a Scenario is to drive the real browser through the CodeceptJS MCP server, query the page to learn locators, and only commit steps that actually pass. This skill is the playbook for that loop.

Two authoring modes, picked by how much of the flow you already know:

- **Mode A — incremental `run_code`** — send one or two `I.*` lines per `run_code`, read the response, repeat. Suited to extending an existing test or porting a manual plan with known steps.
- **Mode B — scaffold-and-pause** (recommended for greenfield / unknown flows) — write a stub `Scenario` containing `I.amOnPage('/...'); pause();`, run via MCP `run_test`. The browser opens and yields control at `pause()` on the live container. Drive the page through `run_code` to discover the flow, then edit the test to replace `pause()` with the verified sequence and re-run.

Both modes share the discovery / locator / commit steps below; the difference is just *where the in-flight exploration happens*.

## Workflow

### 1. Read the project (fundamentals)
Run the **codeceptjs-fundamentals** skill first. You need: active web helper, base URL, plugins (especially `aiTrace`, `auth`), AI provider, page-object names, env vars.

### 2. Map what's already there
Before adding anything, enumerate:
- `npx codeceptjs check -c <config>` — verifies the setup loads (config, helpers, plugins, page objects, hooks, tests, defs). Each pass/fail line doubles as an inventory of what the project has wired up.
- `npx codeceptjs list -c <config>` — every available `I.<method>` for the active helpers, including custom ones. Check before suggesting any method.
- `npx codeceptjs dry-run -c <config>` — every Scenario the active config would load, with naming conventions and tags. Add `--steps` to also print queued `I.*` calls, `--numbers` for step indices.
- MCP equivalents: `list_actions`, `list_tests`.

This catches duplication, surfaces patterns the new test should follow, and confirms a custom step / page-object method doesn't already cover the planned flow.

> ⚠ `dry-run` does **not** initialize plugins. The `auth` plugin's injected `login(...)` (and any other plugin-injected function) is undefined under `dry-run`, so `Before(({ login }) => login('admin'))` raises **"login is not a function"** even though the actual `run` works fine. Use `--steps` to inspect Scenario shape; ignore plugin-inject errors. To verify auth works, do a real `run` (or MCP `run_test`).

### 3. Decide if auth is needed
If the path under test sits behind login, invoke the **codeceptjs-auth** skill:
- If `auth` is already configured, use the existing role: `Before(({ login }) => login())`.
- If not, the auth skill walks through adding the plugin and env-var credentials.
- Public path? Skip.

### 4. Look for similar tests and page objects
Before writing anything new, scan:
- existing Scenarios that touch the same feature area
- page objects in the directories registered under `include`
- custom steps in the actor file
- data factories that already create the entities the test needs (user, post, …)

If a page-object method already encodes the locators for this area, drive through it (`profilePage.openSettings()`) instead of writing raw `I.click` chains.

### 5. Identify the starting page
The page where the new test does its work, **after** any auth. Get a real URL, not a guess. Look for similar tests and page objects. If you don't have that data, ask user where to start. CodeceptJS always use relative urls. The host must be set inside config (helpers.Playwright.url or helpers.WebDriver.url or helpers.Puppeteer.url)

### 6. Make sure aiTrace + MCP are wired
- **aiTrace under MCP** — auto-enabled. The MCP server forces `plugins.aiTrace = { on: 'step', enabled: true }` for every session (`bin/mcp-server.js`), so you do **not** need to add it to the project config when authoring through MCP. The user can override with `start_browser({ plugins: { aiTrace: { enabled: false } } })`.
- **aiTrace for CLI runs** — *not* auto-enabled. Either add `plugins: { aiTrace: { enabled: true } }` to the active config, or pass `-p aiTrace` on the runner. Without it, the verification run in step 10 produces no `output/trace_*/` artifacts and `codeceptjs-run-analysis` has nothing to read.
- **MCP** — confirm the AI client points at `node_modules/codeceptjs/bin/mcp-server.js` with `CODECEPTJS_CONFIG` and `CODECEPTJS_PROJECT_DIR` set. See `node_modules/codeceptjs/docs/mcp.md` if it isn't.
- **Headless** — run tests headlessly by default. Either rely on `setHeadlessWhen(CI)` (export `CI=1` for the session) or set `show: false` in the helper config.

### 7. Open a live session via MCP

Pick a mode:

**Mode A — incremental `run_code`** (existing test extension, known flow):
Send a minimal scaffold to MCP `run_code` — `login(<role>)` if auth is needed, then `I.amOnPage(<starting URL>)`. The response includes URL, ARIA snapshot, screenshot, and console logs. This is the ground truth for everything that follows.

**Mode B — scaffold-and-pause** (greenfield / unknown flow):
Write a draft test stub directly in the test file — minimal but real. For a public page:

```js
Scenario('draft - feature exploration', ({ I }) => {
  I.amOnPage('/')
  pause()
})
```

…or with auth:

```js
Before(({ login }) => login('admin'))

Scenario('draft - feature exploration', ({ I }) => {
  I.amOnPage('/dashboard')
  pause()
})
```

Run it via MCP `run_test`. The browser opens, navigates, and yields control at `pause()` — the response carries `{ status: 'paused', pausedAfter, page, suggestions }`. The same `I` / browser the test is using is now driven by `run_code` against the live page.

### 8. Learn the page and pick locators
Hand off to the **codeceptjs-exploration** skill: read the ARIA snapshot first, fall back to HTML when needed, and use `I.grabWebElement` / `I.grabWebElements` (with permissive XPaths when the obvious locator misses) to enumerate and disambiguate candidates. Commit a locator only after verifying it matches exactly one element via MCP `run_code`. In Mode B, this exploration happens *during the pause window* — the page is sitting there waiting for you.

Translate devtools-style strict locators to readable form **before** they hit the test file — see the **Locators** section below.

### 9. Build the Scenario via MCP

**Mode A** — for each user goal (fill a field, click a button, see a confirmation), run one or two CodeceptJS commands through MCP `run_code`, then read the response.

**Mode B** — work the live page from the pause: try one or two `I.*` lines via `run_code`, read the response, navigate the next step. Each successful command goes into a scratchpad you'll paste back into the test file in step 10. When you've reached the end of the flow you wanted, the scratchpad is your verified sequence.

After every command (either mode) ask:
- Did the URL or ARIA change the way you expected?
- Any new errors in the console logs?
- Does a `grab*` value match what was expected?

If a step fails — try a different locator, add a `waitFor*`, or reconsider the flow. **Stop and ask the user** when something is genuinely ambiguous (two "Save" buttons; an unclear empty state; a feature flag that might not be enabled). Don't push through.

### 10. Commit the verified sequence
When every step has worked once in isolation, paste them into a test file:
- Match existing file naming and the **one-Feature-per-file** rule.
- Use a page-object method or custom step wherever one fits — don't duplicate selectors.
- **Translate every locator to its readable form** (see the **Locators** section). No `{ css: 'input[placeholder*="…"]' }`, no `{ xpath: '//tr[contains(.,…)]//…' }` in committed code. Strict locators are a code-review red flag unless nothing semantic, ARIA, or `locate()`-shaped fits.
- Wrap secrets with `secret(...)`; pull credentials from env vars only.
- Add the tag the suite already uses (`{ tag: '@smoke' }`) when relevant.
- **Mode B specific** — replace the `pause()` line with the verified sequence above it. Remove the `draft` Scenario name and rename to its real intent. Don't leave `pause()` in a committed test.
- Run the file end-to-end with `aiTrace` enabled: `npx codeceptjs run --grep '<scenario name>' --steps`. Hand the result to **codeceptjs-run-analysis** — it reads `output/trace_*/` artifacts via bash tools so you can confirm the flow ran clean. Only declare done when the scenario passes there.

## Locators — readable, semantic, scoped

Pick the highest-level form that fits. Priority, top wins:

1. **Dedicated test attributes** — when the page consistently exposes `data-testid` / `data-qa` / similar. Stable by design.
   ```js
   I.click('[data-testid="submit-order"]')
   ```
2. **Semantic + context** — plain string (label / button text / placeholder / `aria-label`).
   ```js
   I.click('Save', '.toolbar')
   I.fillField('Email', 'u@t.com', '#login-form')
   ```
3. **ARIA role** — survives markup churn; doubles as accessibility check.
   ```js
   I.click({ role: 'button', name: 'Sign In' })
   ```
4. **`locate()` builder** — for structural conditions, repeated row/cell targeting, anything CSS can't express in one line.
   ```js
   I.click(locate('button').withText('Edit').inside(locate('tr').withText('Acme Corp')))
   ```
5. **Strict `{ id }` / `{ name }` / `{ css }`** — when nothing above fits.
   ```js
   I.fillField({ id: 'email' }, 'u@t.com')
   I.seeElement({ css: '.invoice-row.paid' })
   ```
6. **`{ xpath }`** — last resort, for axes (`ancestor`, `following-sibling`) or text predicates the builder doesn't cover.
   ```js
   I.click({ xpath: '//button[@aria-pressed="true"]' })
   ```

If item 1 applies broadly across the app, enable the **`customLocator` plugin** so `$name` resolves to the configured attribute. Once it's on:

```js
// before                                      // after, with customLocator: { attribute: 'data-qa' }
I.click({ css: '[data-qa=submit]' })           I.click('$submit')
I.fillField({ css: '[data-qa=email]' }, ...)   I.fillField('$email', ...)
```

See `node_modules/codeceptjs/docs/plugins.md` § *customLocator*.

### Context

Most actions accept a **context** as the last argument — any locator type. The lookup runs only inside that region, which disambiguates duplicate labels and keeps semantic strings usable. When several actions target the same region, bind the locator once and reuse it; do not repeat the chain inline.

Prefer stable structural regions — they survive UI rewrites better than the markup they wrap:

- Landmark elements — `nav`, `main`, `header`, `footer`, `aside`, `{ role: 'dialog' }`, `{ role: 'navigation' }`.
- App-shell containers — `.main-app`, `.app-content`, `.sidebar`, `.toolbar`, `.modal`, `.drawer`.
- A row, card, or list item identified by its data, expressed via `locate(...)`.

**`I.see` and `I.dontSee` require a context.** Their first argument is plain text matched across the page, so without a context the assertion can resolve against navigation, a footer, or an unrelated component and produce a false pass. Other assertions (`I.seeElement`, `I.seeNumberOfElements`, etc.) take an explicit locator, which scopes them on its own.

### Picking a specific match

When a locator matches several elements, CodeceptJS uses the first by default. To target another match without writing a more specific locator, pass `elementIndex` via `step.opts()` (`import step from 'codeceptjs/steps'`) — accepts a 1-based number, a negative index, or `'first'` / `'last'`. Same path for `step.opts({ exact: true })` to make a single step throw on ambiguity. See `node_modules/codeceptjs/docs/locators.md` § *Picking a specific element*.

Full reference: `node_modules/codeceptjs/docs/locators.md`.

## Waiting

CodeceptJS auto-waits before each action, but explicit waits are still needed when:
- a **loader / spinner / skeleton** must hide before the next step → `I.waitForInvisible('.spinner')`, `I.waitForDetached('.skeleton')`
- a **modal / drawer / panel / section** hasn't rendered yet → `I.waitForVisible('.modal')`, `I.waitForElement({ role: 'dialog' })`
- **data must finish loading** — list rows, cards, charts, async text → `I.waitForElement('.user-row', 10)`, `I.waitForText('Loaded', 10, '.status')`

Detect what to wait for by reading the page HTML / ARIA between MCP steps. If the next element is gated by a spinner overlay or rendered after a fetch, scroll the markup until you find the gating element, pick a stable selector, and wait for the right state (visible, invisible, detached, text-present).

`I.wait(N)` (raw seconds) is OK during **authoring** to confirm a timing hypothesis — if a 5-second sleep makes the step pass, the cause is timing. **Replace it with a specific `I.waitFor*` before committing.** Hardcoded sleeps are slow on fast machines, flaky on slow ones, and hide the real sync point.

## Things to avoid

- Writing tests from imagined locators or imagined routes.
- Defaulting to strict `{ css }` / `{ xpath }` locators when a semantic string, ARIA role, or `locate()` chain would do. See the **Locators** section.
- Hardcoding credentials anywhere — env vars + `secret()` only.
- Skipping the page-object scan.
- Adding `await` to plain action steps (see fundamentals' `await` rule).
- Adding `waitFor*` speculatively before checking whether auto-waiting already handles it.
- Leaving `I.wait(N)` (raw seconds) in committed tests — replace with a specific `waitFor*`.
- Leaving the `pause()` from Mode B's stub in the committed test — it must be replaced with the verified sequence before the file is done.
- Using Mode A when the flow is genuinely unknown — round-tripping `run_code` for every step is slower than Mode B and easier to lose track of state in.
- Declaring the test done without running the committed file end-to-end with `aiTrace` enabled.

## Pointers

- `node_modules/codeceptjs/docs/basics.md` — locators, assertions, waits, hooks
- `node_modules/codeceptjs/docs/test-structure.md` — Feature/Scenario syntax
- `node_modules/codeceptjs/docs/locators.md`, `docs/element-selection.md`
- `node_modules/codeceptjs/docs/pageobjects.md`, `docs/sessions.md`, `docs/within.md`
- `node_modules/codeceptjs/docs/data.md` — REST helper, Data Objects
- `node_modules/codeceptjs/docs/mcp.md` — MCP tool list and client config
- `node_modules/codeceptjs/docs/secrets.md` — `secret()` wrapper
