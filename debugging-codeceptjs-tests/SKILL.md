---
name: debugging-codeceptjs-tests
description: Use when a CodeceptJS 4 test is failing, flaky, or behaving unexpectedly — stack traces from `npx codeceptjs run`, intermittent failures, locator drift, timing issues, "works locally fails in CI". Two debug modes — interactive via the MCP server (`run_step_by_step`, `run_code`) reading per-step traces, or non-interactive via `npx codeceptjs run --debug` (escalate to `DEBUG="codeceptjs:*"` when CodeceptJS internals look at fault — recorder hangs, leaks, plugin races). Don't fix from the error message alone; capture page state and read it. Trigger on broken or flaky tests, run errors, "why does this fail", trace/screenshot/console mentions.
---

# Debugging CodeceptJS 4 Tests

Failures lie. The error usually points at a step that's a side effect of something earlier — auth expired, a frame switch missed, a network call still pending. Reproduce, capture state, and read it before fixing.

Two modes:
- **Interactive** — drive the test step by step through the **MCP server**, inspecting ARIA / HTML / console after each command. Best when you can iterate.
- **Non-interactive** — `npx codeceptjs run --debug` for verbose framework output. Escalate to `DEBUG="codeceptjs:*"` when the *framework itself* looks at fault: recorder hangs, plugin races, event leaks, "step never ran". Best for CI repros, headless servers, framework-level bugs.

## Workflow

### 1. Read the project (fundamentals)
Run the **codeceptjs-fundamentals** skill. You need: helper, plugins on (especially `aiTrace`, `screenshot`, `pageInfo`, `retryFailedStep`, `pause`, `auth`), env vars, whether MCP is wired up. If `aiTrace` is **not** enabled, suggest adding `plugins: { aiTrace: { enabled: true } }` before re-running — most of this skill leans on its output.

### 2. Reproduce minimally
Run only the failing test, with steps printed:
```bash
npx codeceptjs run --grep '<scenario>' --steps
```
Add `--config codecept.ci.conf.js` if the failure is CI-specific. Confirm reproduction before instrumenting further.

### 3. Pick a mode

**Interactive (MCP):**
- `run_step_by_step <test>` — runs the test and writes `output/trace_<TestName>_<hash>/trace.md` plus per-step screenshot, HTML, ARIA, console.
- `run_code <CodeceptJS lines>` — verify a hypothesis against the live page (try a locator, grab a value, navigate). Returns ARIA / URL / screenshot / console.
- `run_test <test>` — subprocess run with JSON reporter, for confirming a fix.
- `list_actions` — sanity-check that an `I.*` method exists on the active helper.

**Non-interactive (CLI):**
- `npx codeceptjs run --grep '<scenario>' --debug` — first move when MCP isn't available. Steps + helper internals + URLs + plugin events.
- `npx codeceptjs run ... --verbose` — adds promise-queue / retry / timeout logs on top of `--debug`.
- `DEBUG="codeceptjs:*" npx codeceptjs run ...` — turns on CodeceptJS's internal debug streams. Reach for this when `--debug` doesn't explain the failure: orphaned timers, event leaks, recorder hangs, plugin races, double-emitted events, "step disappeared from the queue". Narrow with namespaces: `codeceptjs:recorder` (promise queue), `codeceptjs:pause`, `codeceptjs:ai`, `codeceptjs:plugin:<name>`. Most user-level test failures don't need this — it's the framework-internal escape hatch.

### 4. Read the trace
Hand off to **codeceptjs-run-analysis** to walk `output/trace_<TestName>_<hash>/trace.md` and the per-step artifacts. Focus on the **first** failed step — late failures are usually side effects of an earlier silent miss. The run-analysis skill also covers grepping into large HTML, clustering errors across many traces, and comparing reruns when flakiness is in play.

### 5. Form a hypothesis

| Symptom | Likely cause |
|---|---|
| Element missing, page is `/login` | Auth: stale `check`, expired session, missing env var |
| Element in HTML but `display: none` | `waitForVisible`, not `waitForElement` |
| Locator matches 2+ (strict mode) | Disambiguate: ARIA role, `step.opts({ elementIndex })`, `within` |
| Element in screenshot N+1, missing in N | Animation / lazy load — `waitForVisible(loc, t)` |
| 401/403 in console.json | API token expired or env var missing |
| Steps pass, next `I.see` fails | Frame switch missed — wrap in `within({ frame })` |
| Different result CI vs local | `setHeadlessWhen(CI)`, viewport, timing, env var |
| Recorder hangs, step never fires | `DEBUG="codeceptjs:recorder"` to inspect the queue |
| Plugin misbehaves | `DEBUG="codeceptjs:plugin:<name>"` |

### 6. Verify the fix on the live page
Use MCP `run_code` (or `pause()` + `--debug`) to test the candidate fix against the real page **before editing the file**. If it works there, it'll work in the test.

### 7. Apply and re-run
Edit the test, then `npx codeceptjs run --grep '<scenario>' --steps`. Use **codeceptjs-run-analysis** to verify the trace looks right after the fix — and to confirm the failure didn't shift to another step. If the fix introduces a `waitFor*` or `step.opts`, leave a one-line `Why:` comment — those are the comments worth keeping.

## When to reach for which plugin / mode

| You want to … | Use |
|---|---|
| Per-step artifacts after a run | `aiTrace` plugin (`output/trace_*/`) |
| REPL on first failure | `npx codeceptjs run -p pause` (default `on=fail`) |
| Single-step interactively | `npx codeceptjs run -p pause:on=step` |
| Break on a file or URL | `pause:on=file:path=<file>;line=<N>` / `pause:on=url:pattern=<glob>` |
| Step-by-step from an AI agent | MCP `run_step_by_step` |
| Test a hypothesis on the live page | MCP `run_code` or in-test `pause()` |
| Visual replay slideshow | `screenshot:slides=true` → `output/records.html` |
| Auto-suggest fixes for broken locators | `heal` plugin + `--ai` (disabled in `--debug`) |
| Diagnose framework-internal behaviour | `DEBUG="codeceptjs:*"` (or a specific namespace) |
| Inspect specific elements — state, markup, position, children | `I.grabWebElement` / `I.grabWebElements` (cross-helper WebElement API) |
| Drop to native helper APIs when nothing else works | `I.usePlaywrightTo` / `I.usePuppeteerTo` / `I.useWebDriverTo` |

## Waiting (a common cause of flakes)

Most "intermittent" failures are missed waits. Use the trace HTML / ARIA to find the *actual* gating element rather than adding a generic delay:
- a **loader / spinner / skeleton** still on the page → `I.waitForInvisible('.spinner')` / `I.waitForDetached('.skeleton')`
- a **modal / drawer / panel** that hasn't appeared yet → `I.waitForVisible('.modal')` / `I.waitForElement({ role: 'dialog' })`
- async data — list rows, cards, charts, async-rendered text → `I.waitForElement('.user-row', 10)` / `I.waitForText('Loaded', 10, '.status')`

`I.wait(N)` (raw seconds) is fine **during debugging** to confirm a timing hypothesis — if a 5-second sleep makes the test pass, you've found the cause. **Replace it with the specific `I.waitFor*` before committing.** Raw sleeps are slow on fast machines, flaky on slow ones, and hide the real sync point so the next person to touch the test inherits the same problem.

## Inspect the page when the trace isn't enough

When the trace tells you *what* failed but you need more page-state detail to diagnose — "is this button actually disabled?", "are there really two Save buttons?", "what's the rendered markup of this row?" — hand off to the **codeceptjs-exploration** skill. It covers the WebElement API (`I.grabWebElement` / `I.grabWebElements`, state checks, `toSimplifiedHTML`, `toAbsoluteXPath`, iframe walking) and the broad-XPath candidate-discovery technique.

Debug-specific reaches into that toolkit:

- **Button rendered but the click had no effect** — `grabWebElement('Submit')`, then `isEnabled()` + `getBoundingBox()`. Disabled? offscreen? zero-sized?
- **Strict-mode "matched 2 elements"** — exploration's broad-XPath + iterate-and-disambiguate pattern is the canonical fix.
- **Iframe content** — exploration's `inIframe` pattern; the failing step likely needs to be wrapped in `within({ frame })`.

Prefer this over `usePlaywrightTo` / `useWebDriverTo` for inspection: same code across helpers, less boilerplate.

## Native helper API escape hatch

When `I.grabWebElement` and the rest of the `I.*` surface still don't cover it — listening to network requests, manipulating storage, calling a Playwright-only API, raw browser context work — drop down to the underlying helper:

- **Playwright** — `I.usePlaywrightTo('description', async ({ browser, browserContext, page }) => { ... })`
- **Puppeteer** — `I.usePuppeteerTo('description', async ({ page }) => { ... })`
- **WebDriver** — `I.useWebDriverTo('description', async ({ browser }) => { ... })`

The first arg is a label that shows up in step output and traces. The callback receives the helper's native objects. Use these to inspect or manipulate state CodeceptJS doesn't expose — `page.evaluate(() => performance.timing)`, `page.context().cookies()`, `browserContext.on('request', …)`, raw `executeScript` chains. They work inside MCP `run_code` too, so you can poke at internals during a live debug session.

Try the regular `I.*` API first — these escape hatches couple the test to a specific helper. Reach for them only when nothing else works.

## Helper-specific gotchas

- **Playwright** — `strict: true` throws on multi-match. `trace: 'on'` produces `output/trace.zip` (open with `npx playwright show-trace`). Prefer `'load'` / `'domcontentloaded'` over `'networkidle'`.
- **Puppeteer** — `'networkidle0'` can hang on long-polling pages; try `'networkidle2'` or `'domcontentloaded'`.
- **WebDriver** — `smartWait` applies to actions only, not assertions. `executeScript` args must be JSON-serializable.

## Auth-related failures

If the trace shows a redirect to `/login` mid-test, or 401/403 in console, fix **auth**, not the failing step. Check the `auth` plugin's `check`, that credential env vars are exported, and that the cached session under `output/<role>_session.json` isn't stale (delete it to force re-login). The **codeceptjs-auth** skill has the full pattern.

## Things to avoid

- Fixing from the error message without reading the trace.
- Editing the test before verifying the fix in `run_code` — you'll iterate without ground truth.
- Adding `waitFor*` blindly instead of identifying the real gating element from HTML/ARIA.
- Leaving `I.wait(N)` (raw seconds) in committed tests — keep them only while debugging, then replace with the specific `waitFor*`.
- Skipping the config check — `setHeadlessWhen(CI)` or env-driven URLs explain many "works locally fails in CI" reports.
- Hiding the failure with `retries` instead of fixing the cause.

## Pointers

- `node_modules/codeceptjs/docs/mcp.md` — MCP tool list and client config
- `node_modules/codeceptjs/docs/aitrace.md` — plugin config, trace.md format
- `node_modules/codeceptjs/docs/debugging.md` — in-test `pause()`, the `pause` plugin's `on=` modes, IDE setup, DEBUG namespaces
- `node_modules/codeceptjs/docs/heal.md` — self-healing recipes
- `node_modules/codeceptjs/docs/retry.md` — retry semantics across step / scenario / hook
- `node_modules/codeceptjs/lib/plugin/aiTrace.js`, `lib/plugin/pause.js`, `lib/plugin/screenshot.js`, `lib/plugin/browser.js`, `bin/mcp-server.js` — source if docs and code disagree
