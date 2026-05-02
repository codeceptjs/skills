---
name: refactoring-codeceptjs-tests
description: Use when refactoring CodeceptJS 4 tests — cleaning up duplication, extracting page objects, taming long locators, moving raw JS into custom helpers. Works targeted (one test or file) or global (whole tests directory). Always proposes changes before applying. Trigger on phrases like "refactor my tests", "clean up", "extract page object", "this test is too long", "deduplicate", or when reviewing test files for quality.
---

# Refactoring CodeceptJS 4 Tests

Test suites rot in three predictable ways: duplicate UI flows copy-pasted across files, fat locators repeated everywhere, and raw JS escaping into Scenarios. The fix for each is moving the pattern to its proper home — page object, custom helper, or the `auth` plugin — without changing test behaviour.

This skill works **targeted** (refactor one test or file the user named) or **global** (sweep the whole tests directory). Either way it **proposes changes first** and applies after approval, in reviewable batches.

## Workflow

### 1. Read the project (fundamentals)
Run the **codeceptjs-fundamentals** skill. You need: registered page objects (keys under `include`), the actor file (`custom_steps.js` or similar), any custom helpers already configured, and the `auth` plugin's user list. Without this you don't know where extractions land.

### 2. Pick scope
- **Targeted** — a file or Scenario the user named. Read it once, propose, apply.
- **Global** — the configured `tests` glob. Treat as a multi-pass cycle: inventory → propose grouped → approve → apply a batch → re-run affected Scenarios → next batch. Never one big edit.

### 3. Inventory the target
Read the test(s). Note:
- Repeated `I.*` sequences across 2+ Scenarios.
- Locator strings used in 2+ places, or single locators that span multiple lines / deep XPath / nested CSS.
- `I.executeScript` blocks beyond one statement.
- `usePlaywrightTo` / `usePuppeteerTo` / `useWebDriverTo` blocks doing business work (not inspection).
- Hardcoded credentials, URLs, magic strings.
- `I.wait(N)` raw-seconds calls and `await` on plain action steps — fix while you're there.

### 4. Duplicate UI flows → page object methods
Sequences like login form fill, modal-confirm, search-then-click, table-row-edit. Extract to a method named for the user's intent: `loginPage.signIn(email, pass)`, `searchBar.searchFor(term)`, `cartPage.removeItem(name)`. Add to the existing PO when one fits the area; otherwise create a new PO and register it under `include` in the config. Apply the **rule of three** — wait for the second occurrence before extracting; one-off code shouldn't be abstracted.

### 5. Fat locators → `locate()` in page objects
Multi-line XPath, deeply nested CSS, or repeated locator strings. Convert to the `locate(...)` builder, leaning on the new 4.x DSL methods (`withClass`, `withoutClass`, `and`, `andNot`, plus `withText`, `inside`, `withDescendant`). Used once → keep `locate()` inline at the call site. Used 2+ times → store as a named page-object field with `.as('description')` so failures point at a meaningful name.

### 6. Raw JS → custom helper
Big `I.executeScript(...)` blocks, `usePlaywrightTo` doing real DOM walking or business logic, hand-rolled fetches inside a Scenario. Move into a method on a custom helper. Register the helper under `helpers` in the config alongside built-ins. **Critical rule from fundamentals**: inside a custom helper, `I` does **not** exist — compose with another helper via `this.helpers['<Name>']` (e.g. `this.helpers['Playwright'].page.evaluate(...)`, `this.helpers['REST'].sendGetRequest(...)`). Tests stay in the `I.*` vocabulary; helpers expose the new capability.

### 7. Special case — login duplication
If the duplicated UI flow is login, **don't build a `loginPage.signIn` method**. Hand off to the **codeceptjs-auth** skill — that's what the `auth` plugin exists for. Page objects for login flows are a common smell when the project should be using session reuse.

### 8. Cross-page flows (optional)
Site-wide actions used across many Features — `I.acceptCookies()`, `I.goToBilling()`, `I.dismissNotification()` — belong in the actor (`custom_steps.js`), not a single page's PO. The actor is the right home when no single page owns the action.

### 9. Propose, then apply
Group proposals by destination file (`pages/loginPage.js`, `helpers/DbHelper.js`, `tests/checkout_test.js`). Show the user the list before editing. In global mode, get explicit approval and work in batches — three to five files at a time. After each batch, run the affected Scenarios:
```bash
npx codeceptjs run --grep '<scenario or feature>' --steps
```
Hand the result to **codeceptjs-run-analysis** — confirm every affected scenario still passes and no failure shifted to a new step. Refactors that don't run aren't refactors — they're guesses.

## Decision tree

| Pattern | Where it goes |
|---|---|
| UI flow on one page | Method on that page's PO |
| UI flow spanning pages, site-wide | Actor (`custom_steps.js`) |
| Login | `auth` plugin (see `codeceptjs-auth`) |
| Long locator used once | `locate()` inline |
| Long locator used 2+ times | `locate()` stored as a PO field |
| Raw browser API / file system / DB / mail | Custom helper |
| Test-data setup hitting an API | REST helper + Data Object |

## Things to avoid

- Refactoring without re-running the affected Scenarios afterwards.
- Extracting a flow that's only used once — wait for the second occurrence.
- Renaming PO methods or fields without grepping for every caller first.
- Moving code to a custom helper when a page object would do.
- Squashing distinct flows into one method (`loginPage.do(thing)` is a smell).
- Touching Scenario names or tags — CI pipelines and `--grep` filters reference them.
- Mass-applying changes in global mode without batching and re-running.

## Pointers

- `node_modules/codeceptjs/docs/pageobjects.md` — class-based POs, `inject()`, lifecycle hooks
- `node_modules/codeceptjs/docs/custom-helpers.md` — extending `Helper`, the `this.helpers[...]` rule
- `node_modules/codeceptjs/docs/locators.md` — `locate()` builder + 4.x DSL methods
- `node_modules/codeceptjs/docs/best.md` — broader test-organisation guidance
- `codeceptjs-fundamentals` skill — DI, `inject()`, the `I`-unreachable-from-helpers rule
- `codeceptjs-auth` skill — for login deduplication
