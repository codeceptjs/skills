---
name: debugging-codeceptjs-tests
description: Use when a CodeceptJS 4 test is failing, flaky, or behaving unexpectedly — stack traces from `npx codeceptjs run`, intermittent failures, locator drift, timing issues, "works locally fails in CI", "step through this test", "pause at step N", "set a breakpoint". For AI agents the primary path is **MCP with pause** — drop a `pause()` in the test (or pass `pauseAt: N` to `run_test` for a no-edit breakpoint), inspect via `run_code` / `snapshot` against the live browser, release with `continue`. Step indices for `pauseAt` come from `npx codeceptjs dry-run --debug --grep <test> --numbers --no-ansi`. CLI debugging (`npx codeceptjs run --debug`, `DEBUG="codeceptjs:*"`) is the fallback for humans, CI repros, and framework-internal issues (recorder hangs, leaks, plugin races). Don't fix from the error message alone; capture page state and read it. Trigger on broken or flaky tests, run errors, "why does this fail", trace/screenshot/console mentions, breakpoint/pause/step-through requests.
---

# Debugging CodeceptJS 4 Tests

Failures lie. The error usually points at a step that's a side effect of something earlier — auth expired, a frame switch missed, a network call still pending. Reproduce, capture state, and read it before fixing.

Two paths, picked by who's driving:

- **MCP-first (for AI agents)** — drive the test through the MCP server. In-test `pause()` and the `pauseAt: N` option on `run_test` both yield control back to the agent in-process — same `I` / browser the test is using. Inspect via `run_code` / `snapshot`, advance one step at a time via `run_step_by_step` + `continue`, release a pause via `continue`. `aiTrace` artifacts cover the prior steps.
- **CLI fallback (for humans / CI / framework internals)** — `npx codeceptjs run --debug` for verbose framework output. Escalate to `DEBUG="codeceptjs:*"` when the *framework itself* looks at fault: recorder hangs, plugin races, event leaks, "step never ran". Use this path for CI repros, headless servers, and framework-internal bugs.

In-test `pause()` adapts to who's driving: at a TTY it opens the readline REPL; under MCP it yields control to the agent (same in-process `I` / browser); in a non-TTY non-MCP subprocess it prints a notice and resolves immediately so leftover `pause()` calls don't deadlock CI. **Adding `pause()` is now the primary MCP breakpoint** — drop it where you want to look, run via `run_test`, drive the live page through `run_code`, release with `continue`.

## Workflow

### 1. Read the project (fundamentals)
Run the **codeceptjs-fundamentals** skill. You need: helper, plugins on (especially `aiTrace`, `screenshot`, `pageInfo`, `retryFailedStep`, `pause`, `auth`), env vars, whether MCP is wired up. If `aiTrace` is **not** enabled, suggest adding `plugins: { aiTrace: { enabled: true } }` before re-running — most of this skill leans on its output.

### 2. Reproduce minimally
Run only the failing test, with steps printed:
```bash
npx codeceptjs run --grep '<scenario>' --steps
```
Add `--config codecept.ci.conf.js` if the failure is CI-specific. Confirm reproduction before instrumenting further.

### 3. Pick a path

**MCP (primary for AI agents):**
- `run_test <test>` — runs a specific test in-process; shares `I` / browser with `run_code` and `snapshot`. Returns the JSON reporter result on completion, **or** `{ status: 'paused', pausedAfter, page, suggestions }` if the test calls `pause()` or hits the optional `pauseAt: N` breakpoint. From a paused state, drive the live page via `run_code` / `snapshot` and release with `continue`.
- `run_step_by_step <test>` — interactive: pauses after every step. After each `continue`, the test advances one step and re-pauses (or completes). Use when you want to watch the whole flow tick by; use `run_test` with `pauseAt: N` instead for a single targeted breakpoint.
- `continue` — releases a paused test. After `pause()` or `pauseAt`: runs to completion (or to the next `pause()`). After `run_step_by_step`: advances one step.
- `run_code <CodeceptJS lines>` — runs arbitrary CodeceptJS code in the live session (works fresh **and** while a test is paused — same container). Returns the **value the code produced**, captures `console.log` / `info` / `warn` / `error` / `debug` output, and saves a final-state snapshot (URL, ARIA, HTML, screenshot, storage). Use to test a locator hypothesis or grab a value at the failure point.
- `snapshot` — captures current browser state without performing any action (URL, cookies, localStorage, HTML, ARIA, screenshot, console). Use between actions when you want to reason about what to do next without re-running anything.
- `list_actions` — sanity-check that an `I.*` method exists on the active helper.

**CLI (fallback):**
- `npx codeceptjs run --grep '<scenario>' --debug` — first move when MCP isn't available. Steps + helper internals + URLs + plugin events.
- `npx codeceptjs run ... --verbose` — adds promise-queue / retry / timeout logs on top of `--debug`.
- `DEBUG="codeceptjs:*" npx codeceptjs run ...` — turns on CodeceptJS's internal debug streams. Reach for this when `--debug` doesn't explain the failure: orphaned timers, event leaks, recorder hangs, plugin races, double-emitted events, "step disappeared from the queue". Narrow with namespaces: `codeceptjs:recorder` (promise queue), `codeceptjs:pause`, `codeceptjs:ai`, `codeceptjs:plugin:<name>`. Most user-level test failures don't need this — it's the framework-internal escape hatch.

### 4. Set a breakpoint with `pause()` or `pauseAt`

The MCP server installs an in-process pause handler at startup. Whenever a test running through `run_test` hits `pause()` (or completes the `pauseAt: N` step), control yields back to the agent on the same `I` / browser. There's no subprocess, no IPC, and `run_code` / `snapshot` work against the live page — exactly what a paused REPL would give you.

Two ways to land at a breakpoint:

- **In-test `pause()`** — drop `pause()` directly in the test where you want to look. Best when you're already editing the file or want to break inside a `within` / loop / hook.
- **`pauseAt: N` on `run_test`** — programmatic, no test edit required. Pauses after the Nth leaf step completes.

To pick `N`, list the steps with their indices:

```bash
npx codeceptjs dry-run --debug --grep '<scenario>' --numbers --no-ansi
```

Output is one numbered line per leaf step (1-based, per-test). The number on the line you want to stop *after* is the value to pass as `pauseAt`. `--no-ansi` strips colors so the output is clean for parsing.

Once paused (`{ status: 'paused', pausedAfter, page, suggestions }`):

1. **Inspect with `run_code`** — `await I.grabCurrentUrl()`, `await I.grabWebElement(...)`, `await I.seeElement({ role: 'dialog' })`. Each call returns URL + ARIA + console + storage from the live page.
2. **Capture clean state with `snapshot`** between hypotheses — no action, just the artifact bundle.
3. **Walk earlier steps via `aiTrace`** — `output/trace_<TestName>_<hash>/trace.md` has the per-step state for everything that ran before the breakpoint.
4. **Release with `continue`** — runs to completion (or to the next `pause()`). For a step-by-step walk, use `run_step_by_step` instead of `run_test`; each `continue` then advances one step.

### 5. Read the trace
Hand off to **codeceptjs-run-analysis** to walk `output/trace_<TestName>_<hash>/trace.md` and the per-step artifacts. Focus on the **first** failed step — late failures are usually side effects of an earlier silent miss. The run-analysis skill also covers grepping into large HTML, clustering errors across many traces, and comparing reruns when flakiness is in play.

### 6. Form a hypothesis

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

### 7. Verify the fix on the live page
For agents driving MCP, use `run_code` to try the candidate fix in the live session **before editing the file**. If it works there, it'll work in the test. While paused (in-test `pause()` or `pauseAt`), `run_code` operates on the same `I` / browser the test is using, so a candidate replacement step can be tried in place. Humans running with `--debug` at a TTY can use in-test `pause()` for the same purpose at a readline REPL.

### 8. Apply and re-run
Edit the test, then `npx codeceptjs run --grep '<scenario>' --steps`. Use **codeceptjs-run-analysis** to verify the trace looks right after the fix — and to confirm the failure didn't shift to another step. If the fix introduces a `waitFor*` or `step.opts`, leave a one-line `Why:` comment — those are the comments worth keeping.

## When to reach for which plugin / mode

| You want to … | Use |
|---|---|
| Per-step artifacts after a run | `aiTrace` plugin (`output/trace_*/`) |
| REPL on first failure | `npx codeceptjs run -p pause` (default `on=fail`) |
| Single-step interactively | `npx codeceptjs run -p pause:on=step` |
| Break on a file or URL | `pause:on=file:path=<file>;line=<N>` / `pause:on=url:pattern=<glob>` |
| Programmatic breakpoint at step N (no test edit) | MCP `run_test` with `pauseAt: N` (discover N via `dry-run --numbers`) |
| In-test breakpoint at a specific line | drop `pause()` in the test, then MCP `run_test` |
| Step-by-step REPL from an AI agent | MCP `run_step_by_step`, then `continue` between steps |
| Release a paused test | MCP `continue` |
| Test a hypothesis on the live page (agent) | MCP `run_code` (works fresh **and** while paused) |
| Capture state without acting (agent) | MCP `snapshot` |
| Test a hypothesis on the live page (human, TTY) | in-test `pause()` + `npx codeceptjs run --debug` |
| List steps with their indices (for `pauseAt`) | `npx codeceptjs dry-run --debug --grep '<test>' --numbers --no-ansi` |
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
- Committing `pause()` calls. They're a debugging tool — remove (or replace with `pauseAt`) before merging. A `pause()` left in a test that runs in a non-TTY non-MCP CI subprocess will print a notice and skip, but it's still noise on every run.
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
