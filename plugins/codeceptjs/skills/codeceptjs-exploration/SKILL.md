---
name: codeceptjs-exploration
description: Use to explore a page in CodeceptJS — read its ARIA tree, inspect candidate elements, pick a stable locator. Drives the live browser through MCP `run_code`, prefers ARIA over HTML, uses `I.grabWebElement` / `I.grabWebElements` with permissive XPaths to enumerate candidates and `toSimplifiedHTML` / `toAbsoluteXPath` to disambiguate. Other skills (`writing-codeceptjs-tests`, `debugging-codeceptjs-tests`, `refactoring-codeceptjs-tests`) invoke this whenever they need to learn what's on a page.
---

# CodeceptJS Page Exploration

Authoring a test, debugging a failure, and refactoring a stale locator all share one task: open a page, find the right element, pick a stable locator. This skill is the playbook.

## How to look at a page

Drive everything through MCP. Two tools matter for exploration:

- **`run_code`** — runs CodeceptJS code and returns the value the code produced, captures `console.*` output, and saves a final-state snapshot. Use when you want to *do something and look at the result* (try a locator, grab a value, navigate).
- **`snapshot`** — captures the current state without performing any action: URL, cookies, localStorage, HTML, ARIA, screenshot, console. Use when you want to look at "what's on the page right now" between two actions, without re-running anything.

Three artifact sources, in order of preference:

1. **ARIA snapshot first.** Structured, free of styling noise, easy to scan for duplicates and accessibility names.
2. **Screenshot.** Visual confirmation — catches layout breaks, missing icons, "rendered but wrong" cases that ARIA can't show.
3. **HTML / outer markup.** Pull only when ARIA is missing crucial context: custom widgets without accessible names, attribute-driven behaviour, dynamic content with no roles.

## Inspect a known element

`I.grabWebElement(locator)` returns one WebElement; `I.grabWebElements(locator)` returns an array. Same cross-helper API on Playwright / Puppeteer / WebDriver, returns values back through MCP `run_code`.

| You want to … | Method on WebElement |
|---|---|
| Confirm rendered / visible / enabled | `exists()`, `isVisible()`, `isEnabled()` |
| Read text / value / attribute / property | `getText()`, `getValue()`, `getAttribute(n)`, `getProperty(n)` |
| Where is it on the page | `getBoundingBox()` — flags offscreen / zero-sized |
| The actual rendered markup | `toOuterHTML()`, `toSimplifiedHTML(300)` (truncated, MCP-friendly) |
| Stable selector for a fix | `toAbsoluteXPath()` |
| Look inside an iframe | `inIframe(async (body) => { ... })` |
| Drill into children | `$(loc)`, `$$(loc)` |
| Run a browser-side function | `evaluate(fn, ...args)` |

## Discover candidates when the obvious locator misses

When `Edit` matches nothing, the control might say "Change", carry `aria-label="Edit user"`, or live in a `.btn-edit` class. Cast a wide net: pass a permissive XPath to `I.grabWebElements`, then disambiguate.

Build the XPath by ORing:
- visible text — `text()` (or `.` to match descendants too)
- relevant attributes — `@class`, `@aria-label`, `@title`, `@data-action`, `@id`
- **synonyms** — "edit" / "change" / "modify"; "delete" / "remove" / "trash"; "submit" / "send" / "save"

Wrap each match with `translate(...)` for case-insensitive `contains`:

```
//*[contains(translate(., 'EDIT', 'edit'), 'edit')
    or contains(translate(@class, 'EDIT', 'edit'), 'edit')
    or contains(translate(@aria-label, 'EDIT', 'edit'), 'edit')
    or contains(translate(., 'CHANGE', 'change'), 'change')]
```

Then iterate `toSimplifiedHTML(150)` over the result, review the candidates, pick the right one, and commit a stable locator built from its discriminating attribute or text — or `toAbsoluteXPath()` if nothing else is stable.

## Pick a stable locator

Once the right element is identified, choose the locator with the highest semantic value that is still unique:

1. ARIA — `{ role: 'button', name: 'Edit user' }`. Survives CSS refactors.
2. Visible label / semantic text — `'Edit user'`. Easy to read.
3. `[data-testid="edit-user"]` if the team uses test attributes.
4. Composed CSS with a stable parent — `#user-row-42 button.edit`.
5. `toAbsoluteXPath()` from the candidate review — last resort; flag for the team to add a `data-testid`.

Don't commit a locator you didn't verify. After picking, run `I.seeElement(<locator>)` (or `grabWebElement(<locator>)`) through MCP `run_code` to confirm it matches exactly one element.

## Common patterns

- **Strict mode found 2 matches** — `grabWebElements('Save')`, `toSimplifiedHTML(200)` each, find a discriminator (parent class, `data-*`, surrounding text).
- **Button rendered but doesn't act** — `grabWebElement('Submit')`, then `isEnabled()` + `getBoundingBox()` — disabled? offscreen? zero-sized?
- **Wrong row in a list** — `grabWebElements('.user-row')`, `getText()` per row, identify, then `getAttribute('data-id')` for a stable hook.
- **Inside an iframe** — `(await I.grabWebElement('iframe.editor')).inIframe(async (body) => body.$('button'))`.
- **No semantic name on the element** — `toAbsoluteXPath()` for now; flag the team to add a `data-testid`.

## Things to avoid

- Choosing a locator without seeing the candidates first — you'll guess wrong and the test will be flaky.
- Committing `toAbsoluteXPath()` when a semantic locator is right there.
- Ignoring the screenshot — "element exists in HTML" doesn't mean "user can see it".
- Reaching for `usePlaywrightTo` / `useWebDriverTo` when WebElement methods cover the case (cross-helper code is preferred).

## Pointers

- `node_modules/codeceptjs/docs/web-element.md` — full WebElement API
- `node_modules/codeceptjs/docs/locators.md` — locator strategies and priorities
- `node_modules/codeceptjs/docs/element-selection.md` — `step.opts({ elementIndex })`, strict mode
- `node_modules/codeceptjs/docs/mcp.md` — MCP tool list
- `codeceptjs-fundamentals` skill — locator priority and the `await` rule
