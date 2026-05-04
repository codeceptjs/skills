---
name: codeceptjs-run-analysis
description: Use after running CodeceptJS tests with the `aiTrace` plugin enabled — analyse the trace artifacts (trace.md, per-step HTML/ARIA/screenshots, console logs) via bash tools. Toolkit, not a workflow — use cases include verifying a fix, clustering errors across a CI fail-storm, diagnosing flakiness across reruns, or investigating a single failure. Other skills (`writing-codeceptjs-tests`, `debugging-codeceptjs-tests`, `refactoring-codeceptjs-tests`) invoke this whenever a run has happened and needs to be reviewed. Trigger on phrases like "what failed", "analyse the run", "cluster these errors", "is it flaky", "did the fix hold".
---

# CodeceptJS Run Analysis

After `npx codeceptjs run`, the trace artifacts land in `output/`. This skill is the playbook for reading them efficiently — pulling out the right step, the right file, and the right slice of a giant HTML snapshot using bash tools rather than re-running the test through MCP.

There's no single end goal — pick the use case that matches the situation. The foundations (where artifacts live, how to read them, what tools to reach for) apply to all of them.

## Foundations

### Ensure aiTrace is on
The whole skill leans on `output/trace_<TestName>_<hash>/trace.md` and its sibling artifacts. Confirm `plugins: { aiTrace: { enabled: true } }` in the active config (run `codeceptjs-fundamentals` if you don't already know). Without aiTrace there are screenshots and `pageInfo` dumps at most — useful but partial; suggest enabling and re-running before deep analysis.

The MCP `run_step_by_step` tool is now interactive (pauses after every step; agent advances via `continue`) and no longer auto-writes a per-run artifact bundle. `aiTrace` is the only source of per-step trace files now — for ad-hoc `run_code` and `snapshot` calls you still get a single-shot artifact set under `output/trace_run_code_*` / `output/snapshot_*`.

### Locate traces
Trace directories are `output/trace_<TestName>_<short-hash>/`; reruns produce a new dir per run. **When the right trace isn't obvious, sort by modification time and take the most recent** (`ls -dt`) — almost always the run you just kicked off.

### Read trace.md first
trace.md is the index — each step block links to its screenshot/HTML/ARIA/console. **Focus on the step marked failed; if no failure is explicitly marked, jump to the last step in the trace** — execution usually stops where the failure happened. When more than one step shows failed, the **first** is usually the cause; the rest are cascading side effects.

### Open artifacts in the right order
For the focus step `NNNN_<step>`:

| Artifact | When |
|---|---|
| `NNNN_*_aria.txt` | First read — leaner than HTML, structured, easy to scan for duplicates |
| `NNNN_*_screenshot.png` | Visual confirmation — layout, animation, "rendered but wrong" |
| `NNNN_*_page.html` | Only when ARIA is missing context. **Use `grep`, not `cat`** |
| `NNNN_*_console.json` | JS errors, 4xx/5xx, deprecation warnings explaining "vanished" elements |
| `NNNN_*_storage.json` | Cookies + localStorage at this step. First place to look when auth is the suspected culprit. |

### Never read big files whole
HTML snapshots can be megabytes; `console.json` arrays can be long.

- Use `grep` to search HTML for expected elements / attributes / locator variants (text, class, aria-label, `data-*`). Line numbers and context flags make the match readable.
- Use `jq` to filter `console.json` for errors or specific event types instead of reading every entry.
- When scanning many files, `grep -l` returns filenames-only.

## Use cases

### Verify a fix held
After editing a test (e.g. via MCP) and re-running it via CLI: locate the latest trace, read trace.md, confirm no FAILED markers. Glance at `console.json` for warnings worth fixing while you're there.

### Cluster errors across a CI batch
When many tests failed in one run: extract the failing-step lines from every `trace.md` in the batch, group by signature (`grep` + `sort` + `uniq -c`), rank by frequency.

- **Same error in many tests = systemic.** Env var missing, auth broken, base URL wrong, deploy regression. Fix the root cause once and rerun the batch.
- **Different errors per test = local issues.** Triage one at a time.

Pick the most-frequent root cause first; rerun to see how many tests came back along with it.

### Diagnose flakiness across reruns
Run the same test 5–10 times. Extract which step failed in each trace and look at how the answer varies. If the failing step is the *same* but the surrounding state differs, `diff` the ARIA snapshots at that step between two runs to spot what changed.

- **Different step each run** → timing, environment, external service.
- **Same step, different reason** → likely a missing or wrong wait (see Waiting in writing/debug skills).
- **`console.json` differs between runs** → transient backend / network errors.
- **Bounding box differs** → layout reflow or late-loading content shifting things.

### Investigate a single failure
Locate the trace, jump to the failed-or-last step, open ARIA + screenshot first, console.json second, HTML last (and only via `grep`). Form a hypothesis. If the trace alone isn't enough — the failure resists static analysis or the page state needs live poking — hand off to **debugging-codeceptjs-tests** for the MCP-driven loop.

## After the analysis

- **Systemic cause across many tests** — fix the root once (env, auth, deploy, base URL). Not per test.
- **Locator drift** — invoke **codeceptjs-exploration** to pick a new stable locator.
- **Timing / wait issue** — apply the Waiting guidance from the writing/debugging skills, replacing `I.wait(N)` with a specific `waitFor*`.
- **Failure resists static analysis** — invoke **debugging-codeceptjs-tests** for the live MCP loop.
- **Test passed cleanly** — done. Still glance at `console.json` for warnings worth fixing now.

## Things to avoid

- Reading large HTML or `console.json` files whole — search them with `grep` / `jq` instead.
- When more than one step is marked failed, stopping at the last one — the **first** failure is usually the cause and the rest are cascading.
- Triaging individual failures before clustering — fixing 12 symptoms of one bug is wasted effort.
- Drawing flakiness conclusions from a single run — needs 5+ reruns.
- Deleting `output/` mid-investigation; the artifacts are the only record of the run.

## Pointers

- `node_modules/codeceptjs/docs/aitrace.md` — trace format and config knobs
- `node_modules/codeceptjs/docs/debugging.md` — verbose flags, `pause` plugin's `on=` modes
- `node_modules/codeceptjs/docs/reports.md` — alternative reporters
- `codeceptjs-fundamentals` — what's actually configured
- `codeceptjs-exploration` — when locator drift is the cause
- `debugging-codeceptjs-tests` — when post-mortem isn't enough
