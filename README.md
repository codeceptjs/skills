# CodeceptJS Skills

AI agent skills for working with [CodeceptJS 4](https://codecept.io). Eight skills covering project orientation, authoring, debugging, refactoring, and CI auto-repair — all written against the [Agent Skills](https://agentskills.io) open standard.

## Install

### Claude Code (plugin marketplace)

From inside Claude Code:

```text
/plugin marketplace add codeceptjs/skills
/plugin install codeceptjs@codeceptjs-skills
```

That registers this repo as a marketplace and installs the `codeceptjs` plugin — all 8 skills, ready to invoke as `/codeceptjs-fundamentals`, `/writing-codeceptjs-tests`, etc.

### Any other supported agent (`npx skills`)

For Cursor, OpenAI Codex, GitHub Copilot, VS Code, Goose, OpenHands, Junie, Gemini CLI, and [many more](https://agentskills.io), use the [`skills`](https://skills.sh) CLI:

```bash
npx skills add codeceptjs/skills
```

The CLI runs an interactive menu — pick which skills to install and whether to install globally (across all your projects) or only in the current project. Update later with `npx skills update`.

### Manual install

If you can't run the CLI (locked-down environment, custom layout, etc.), each `<skill-name>/SKILL.md` at the repo root is a plain Markdown file you can drop into the agent's skills directory by hand:

- **Claude Code** — `.claude/skills/<skill-name>/SKILL.md` (project) or `~/.claude/skills/<skill-name>/SKILL.md` (personal). [Docs](https://code.claude.com/docs/en/skills).
- **Cursor** — `.cursor/skills/<skill-name>/SKILL.md`. [Docs](https://cursor.com/docs/context/skills).
- **OpenAI Codex** — under the project's skills directory. [Docs](https://developers.openai.com/codex/skills/).
- **GitHub Copilot / VS Code** — Agent Skills supported natively. [Copilot docs](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills), [VS Code docs](https://code.visualstudio.com/docs/copilot/customization/agent-skills).
- **AGENTS.md tools** (Aider, Cline, etc.) — concatenate every `SKILL.md` into a single `AGENTS.md` at repo root.

## Skills

| Skill | Use when |
|---|---|
| [`codeceptjs-fundamentals`](./codeceptjs-fundamentals/SKILL.md) | **Always run first.** Compact primer on CodeceptJS internals (helpers, `I`, DI container, `inject()`, custom helpers, plugins as hook listeners, the `await` rule), then reads the project's config and reports the active helper, plugins, env switching, and page objects. |
| [`codeceptjs-auth`](./codeceptjs-auth/SKILL.md) | A test needs login or different user roles. Configures the `auth` plugin from the real login page HTML, with env-var credentials and `secret()`-wrapped passwords. |
| [`codeceptjs-exploration`](./codeceptjs-exploration/SKILL.md) | Need to look at a page — read ARIA, inspect candidate elements, pick a stable locator. Covers the WebElement API and the broad-XPath candidate-discovery technique. Invoked by writing, debugging, and refactoring. |
| [`codeceptjs-run-analysis`](./codeceptjs-run-analysis/SKILL.md) | After `codeceptjs run` — read `output/trace_*/` artifacts via bash tools to verify a fix, cluster errors across many failures, or diagnose flakiness across reruns. Invoked by writing, debugging, refactoring, and ci-fix. |
| [`writing-codeceptjs-tests`](./writing-codeceptjs-tests/SKILL.md) | Authoring or extending tests. Two modes — incremental `run_code` for known flows, scaffold-and-pause (`I.amOnPage(...); pause();`, drive the live page from MCP, replace `pause()` with the verified sequence) for greenfield ones. Learns locators from ARIA, commits the verified sequence. |
| [`debugging-codeceptjs-tests`](./debugging-codeceptjs-tests/SKILL.md) | A test is failing or flaky. Primary path: drop `pause()` in the test (or pass `pauseAt: N` to MCP `run_test`, with N from `dry-run --numbers`), inspect via `run_code` / `snapshot`, release with `continue`. CLI fallback (`--debug`, `DEBUG="codeceptjs:*"`) for humans, CI repros, and framework-internal issues. |
| [`refactoring-codeceptjs-tests`](./refactoring-codeceptjs-tests/SKILL.md) | Cleaning up an existing suite — extracting page objects, taming long locators, moving raw JS into custom helpers. Targeted (one file) or global. Proposes before applying. |
| [`ci-fix-tests`](./ci-fix-tests/SKILL.md) | Non-interactive CI mode: a run failed; auto-attempt safe fixes (locator drift, missing waits), rerun only the failing scenarios, roll back if no improvement, always write `output/ci-fix.md`. |

`codeceptjs-fundamentals`, `codeceptjs-exploration`, `codeceptjs-run-analysis`, and `codeceptjs-auth` are utility skills that the four action skills (writing, debugging, refactoring, ci-fix) invoke as needed. Users typically trigger an action skill; the utility skills get pulled in automatically.

## How Agent Skills work

Each skill is a folder containing a `SKILL.md` file with YAML frontmatter (`name`, `description`) and markdown instructions. Agents use **progressive disclosure**:

1. **Discovery** — at startup the agent loads only each skill's name and description.
2. **Activation** — when a task matches a skill's description, the agent pulls the full `SKILL.md` into context.
3. **Execution** — the agent follows the instructions, optionally running bundled scripts or loading referenced files.

That keeps the upfront context small while letting one project ship many skills. The format was originally developed by Anthropic, released as an open standard, and is now supported across the agent ecosystem — see [agentskills.io](https://agentskills.io) for the full client list and the format spec.

## Format

```
<skill-name>/
└── SKILL.md
```

Each `SKILL.md` starts with frontmatter:

```yaml
---
name: <skill-name>
description: One sentence describing what the skill does and when it should trigger.
---
```

…followed by markdown body. `description` is the field every supported agent relies on for triggering — front-load the key use case, since most tools cap descriptions around 1–1.5 KB before truncation.

Conventions used in this repo:

- Sub-150 lines per skill; concept-driven prose over code dumps.
- Pointers to `node_modules/codeceptjs/docs/<file>.md` rather than reproducing content — the skills work in any project that has CodeceptJS installed.
- Cross-references between skills by name (`see codeceptjs-exploration`) instead of duplicating content.
- No `await` on plain action steps in any code snippet (matches CodeceptJS's `await` rule).
- Credentials always come from env vars and pass through `secret()`.

## Contributing

Before adding a new skill:

1. Read every existing `SKILL.md` — they share a tone and an opinion about brevity.
2. Check whether your idea is already covered by one of them. If you'd be duplicating triggers, extend the existing skill instead.
3. Toolkits (no single end goal) belong in a "foundations" + "use cases" structure — see `codeceptjs-run-analysis` for the pattern. Workflows (one goal) use numbered steps — see `ci-fix-tests`.

When you add or rename a skill, update the table above.
