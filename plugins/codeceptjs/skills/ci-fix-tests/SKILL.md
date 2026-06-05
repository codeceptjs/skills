---
name: ci-fix-tests
description: "Use on CI in non-interactive mode when a CodeceptJS run failed — automatically attempt safe fixes (locator drift, missing waits), rerun only the failing scenarios, compare against the baseline, roll back any edit that didn't help, and always write a markdown report at `output/ci-fix.md` for the CI step to consume. Conservative by design: no refactors, no config edits, no auth fixes, no flake-masking retries. Trigger on \"ci fix tests\", \"auto-fix failing tests\", \"attempt repair on CI\", or as a CI step after a failed run."
---

# Auto-fix CodeceptJS Tests on CI

A non-interactive auto-repair loop. Reads the failed-tests baseline from `output/` (where `aiTrace` left trace artifacts), attempts a small set of safe fixes, reruns only the failing scenarios, compares, and writes a report at `output/ci-fix.md`. Rolls back any edit that didn't help.

**Conservative by design.** CI is not the place to restructure tests, change abstractions, or fix anything that needs human judgement. The goal is "pass the run". Anything riskier is escalated through the report.

This skill assumes a working **git** checkout — rollback uses `git checkout -- <file>` to restore originals.

## Workflow

### 1. Read the baseline failures
Use the **codeceptjs-run-analysis** skill against the existing `output/trace_*/` artifacts to enumerate every failing scenario. Record:

- the **set** of failing tests (file path + scenario name)
- the failed step in each, and the locator / wait / assertion involved
- a cluster signature per failure (so unique vs systemic is clear)

Save this as the **baseline** — `count`, `failing_set`, `clusters`.

### 2. Pick safe fix candidates
Only attempt fixes from this allowed list:

- **Locator drift** — the failed locator no longer matches anything but a similar element exists. Use the **codeceptjs-exploration** skill (headless) to find candidates; pick one with high semantic stability (ARIA `{ role, name }` → visible text → `data-testid` → composed CSS). Replace the locator at the failing step only.
- **Missing wait for a spinner / loader / modal** — the failed step's ARIA snapshot shows a spinner or skeleton present, or the target element appears later. Add a single matching `I.waitFor*` immediately before the failing step.
- **`I.wait(N)` replacement** — when a hardcoded sleep is the only thing between a failing assertion and a passing one and the gating element is identifiable, replace the sleep with a specific `waitFor*`.

**Do not attempt:**
- Auth / credential / env-var fixes — those are CI configuration, not test code.
- Refactoring (extracting page objects, custom helpers, splitting scenarios).
- Renaming Scenarios or tags — CI's `--grep` and pipelines reference them.
- Adding `retries: N` to a Scenario — masking flake is a human decision.
- Anything outside the failing test file (no `codecept.conf.*`, no helper, no PO edits).

If a failure doesn't match any allowed-fix pattern, **skip it** and record it as `unresolved` for the report.

### 3. Apply edits and rerun
Track every modification: file path, line number, before/after snippet, fix kind. Then rerun **only the failing scenarios** under the same config CI used:

```bash
npx codeceptjs run --grep '<scenario>|<scenario>|...'
```

Find the right config via `package.json` scripts or the CI workflow file. Match the worker count if `run-workers` was used originally.

### 4. Compare against the baseline
Use **codeceptjs-run-analysis** again to enumerate the failing set after the fix.

| New failing-set vs baseline | Status |
|---|---|
| Empty (zero failures) | `SUCCESS_FULL` |
| Strict subset of the baseline | `SUCCESS_PARTIAL` |
| Identical set | `NO_IMPROVEMENT` |
| Larger, or contains a test not in baseline | `REGRESSED` |

### 5. Decide: keep or roll back

- `SUCCESS_FULL` / `SUCCESS_PARTIAL` → **keep** the edits.
- `NO_IMPROVEMENT` / `REGRESSED` → **roll back every edit** with `git checkout -- <files>`. The on-disk state at the end of the skill must be identical to what it was at the start.

A regression is treated the same as no improvement — the attempt didn't work, the working state is what was on disk before. Don't try to "undo only the bad ones" — the whole batch goes.

### 6. Write the report (always)
Write `output/ci-fix.md` regardless of outcome — CI reads it to surface results.

## Report format

```markdown
# CI Fix Report — <ISO timestamp>

**Status:** SUCCESS_FULL | SUCCESS_PARTIAL | NO_IMPROVEMENT | REGRESSED

## Baseline
Failing scenarios: N
- `tests/foo_test.js` — "user can log in"
- `tests/bar_test.js` — "checkout flow"
- ...

## Attempted fixes
- `tests/foo_test.js:42` — locator drift: `'Save'` → `{ role: 'button', name: 'Save' }`
- `tests/bar_test.js:15` — added `I.waitForInvisible('.spinner')` before checkout click
- `tests/baz_test.js:7` — replaced `I.wait(3)` with `I.waitForVisible('.confirmation-dialog', 10)`

## Result
Failing scenarios after fix: M (was N)
- Resolved: `tests/foo_test.js` — "user can log in"
- Still failing: `tests/bar_test.js` — "checkout flow"
- New failures (REGRESSED only): `tests/baz_test.js` — "edit profile"

## Failure clusters (NO_IMPROVEMENT / REGRESSED only)
- **Cluster A (3 scenarios)** — `Element not found: button.btn-primary`. Likely a deploy regression or design change.
- **Cluster B (1 scenario)** — `Navigation timeout to /checkout`. Possible backend issue.

## Discarded changes (NO_IMPROVEMENT / REGRESSED only)
- `tests/foo_test.js` — restored
- `tests/bar_test.js` — restored
- `tests/baz_test.js` — restored
```

The first line of `Status:` is the machine-parseable signal. The rest is for the human reading the failed CI build.

## Things to avoid

- Editing files outside the failing test files. Config, helpers, page objects, and CI workflows are off-limits in this mode.
- Attempting more than one fix kind on the same scenario in one pass. Fix one thing, observe, then try another.
- `retries: N` to mask flakiness — human decision.
- Skipping rollback on `NO_IMPROVEMENT`. Even no-op edits can confuse the next run or the next reviewer.
- Writing a partial or missing `output/ci-fix.md`. CI depends on it; the absence of a report is itself a failure mode.
- Running the **whole** suite for the verification step — only the originally-failing scenarios, to keep CI time bounded.

## Pointers

- `codeceptjs-run-analysis` — read trace artifacts, cluster failures, build the baseline set.
- `codeceptjs-exploration` — find replacement locators when one has drifted.
- `codeceptjs-fundamentals` — confirm helper, config, which env the CI run used.
- `node_modules/codeceptjs/docs/aitrace.md` — trace format.
