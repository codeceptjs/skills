---
name: writing-codeceptjs-tests
description: Use when writing a new CodeceptJS 4 test, extending an existing Scenario, or porting a manual test plan to code. Builds tests live — opens the real page through the CodeceptJS MCP server, queries ARIA/HTML to learn locators, runs each step incrementally to verify it works, then commits the verified sequence to a test file. Never invents locators or flows from imagination; drives the actual browser. Trigger on any request to create, write, add, draft, or scaffold a CodeceptJS test, login flow, or end-to-end check.
---

# Writing CodeceptJS 4 Tests

A test that was never executed during authoring is unreliable by definition. The right way to write a Scenario is to drive the real browser one step at a time through the CodeceptJS MCP server, query the page to learn locators, and only commit steps that actually pass. This skill is the playbook for that loop.

## Workflow

### 1. Read the project (fundamentals)
Run the **codeceptjs-fundamentals** skill first. You need: active web helper, base URL, plugins (especially `aiTrace`, `auth`), AI provider, page-object names, env vars.

### 2. List existing tests
See what's already there — naming conventions, tags in use, typical Feature granularity:
- CLI: `npx codeceptjs list` or list test files in the configured glob
- MCP: `list_tests`

This catches duplication and surfaces patterns the new test should follow.

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
- **aiTrace** — ensure `plugins: { aiTrace: { enabled: true } }` is in the active config; without it, every failure costs another reproduction round-trip.
- **MCP** — confirm the AI client points at `node_modules/codeceptjs/bin/mcp-server.js` with `CODECEPTJS_CONFIG` and `CODECEPTJS_PROJECT_DIR` set. See `node_modules/codeceptjs/docs/mcp.md` if it isn't.
- **Headless** — run tests headlessly by default. Either rely on `setHeadlessWhen(CI)` (export `CI=1` for the session) or set `show: false` in the helper config.

### 7. Open the starting page via MCP
Send a minimal scaffold to MCP `run_code` — `login(<role>)` if auth is needed, then `I.amOnPage(<starting URL>)`. The response includes URL, ARIA snapshot, screenshot, and console logs. This is the ground truth for everything that follows.

### 8. Learn the page and pick locators
Hand off to the **codeceptjs-exploration** skill: read the ARIA snapshot first, fall back to HTML when needed, and use `I.grabWebElement` / `I.grabWebElements` (with permissive XPaths when the obvious locator misses) to enumerate and disambiguate candidates. Commit a locator only after verifying it matches exactly one element via MCP `run_code`.

### 9. Build the Scenario step by step via MCP
For each user goal (fill a field, click a button, see a confirmation), run one or two CodeceptJS commands through MCP `run_code`, then read the response. After every command ask:
- Did the URL or ARIA change the way you expected?
- Any new errors in the console logs?
- Does a `grab*` value match what was expected?

If a step fails — try a different locator, add a `waitFor*`, or reconsider the flow. **Stop and ask the user** when something is genuinely ambiguous (two "Save" buttons; an unclear empty state; a feature flag that might not be enabled). Don't push through.

### 10. Commit the verified sequence
When every step has worked once in isolation, paste them into a test file:
- Match existing file naming and the **one-Feature-per-file** rule.
- Use a page-object method or custom step wherever one fits — don't duplicate selectors.
- Wrap secrets with `secret(...)`; pull credentials from env vars only.
- Add the tag the suite already uses (`{ tag: '@smoke' }`) when relevant.
- Run the file end-to-end outside the MCP sandbox: `npx codeceptjs run --grep '<scenario name>' --steps`. Hand the result to **codeceptjs-run-analysis** — it reads `output/trace_*/` artifacts via bash tools so you can confirm the fix held without re-running through MCP. Only declare done when the scenario passes there.

## Waiting

CodeceptJS auto-waits before each action, but explicit waits are still needed when:
- a **loader / spinner / skeleton** must hide before the next step → `I.waitForInvisible('.spinner')`, `I.waitForDetached('.skeleton')`
- a **modal / drawer / panel / section** hasn't rendered yet → `I.waitForVisible('.modal')`, `I.waitForElement({ role: 'dialog' })`
- **data must finish loading** — list rows, cards, charts, async text → `I.waitForElement('.user-row', 10)`, `I.waitForText('Loaded', 10, '.status')`

Detect what to wait for by reading the page HTML / ARIA between MCP steps. If the next element is gated by a spinner overlay or rendered after a fetch, scroll the markup until you find the gating element, pick a stable selector, and wait for the right state (visible, invisible, detached, text-present).

`I.wait(N)` (raw seconds) is OK during **authoring** to confirm a timing hypothesis — if a 5-second sleep makes the step pass, the cause is timing. **Replace it with a specific `I.waitFor*` before committing.** Hardcoded sleeps are slow on fast machines, flaky on slow ones, and hide the real sync point.

## Things to avoid

- Writing tests from imagined locators or imagined routes.
- Hardcoding credentials anywhere — env vars + `secret()` only.
- Skipping the page-object scan.
- Adding `await` to plain action steps (see fundamentals' `await` rule).
- Adding `waitFor*` speculatively before checking whether auto-waiting already handles it.
- Leaving `I.wait(N)` (raw seconds) in committed tests — replace with a specific `waitFor*`.
- Declaring the test done without running the committed file end-to-end.

## Pointers

- `node_modules/codeceptjs/docs/basics.md` — locators, assertions, waits, hooks
- `node_modules/codeceptjs/docs/test-structure.md` — Feature/Scenario syntax
- `node_modules/codeceptjs/docs/locators.md`, `docs/element-selection.md`
- `node_modules/codeceptjs/docs/pageobjects.md`, `docs/sessions.md`, `docs/within.md`
- `node_modules/codeceptjs/docs/data.md` — REST helper, Data Objects
- `node_modules/codeceptjs/docs/mcp.md` — MCP tool list and client config
- `node_modules/codeceptjs/docs/secrets.md` — `secret()` wrapper
