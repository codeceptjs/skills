# CodeceptJS 4 Skills

Skills for AI coding assistants working on CodeceptJS 4 projects. Each skill is a single `SKILL.md` file with YAML frontmatter (`name`, `description`) and a markdown body following the [Agent Skills](https://agentskills.io) open standard.

The skills assume CodeceptJS 4.x (ESM) and lean on plugins introduced in 4.x: `aiTrace`, `pauseOn`, the `auth` plugin, the WebElement API (`grabWebElement` / `grabWebElements`), and the MCP server at `bin/mcp-server.js`. They reference canonical docs at `node_modules/codeceptjs/docs/...` rather than duplicating them, so they stay short and work in any project that has CodeceptJS installed.

## What's in here

| Skill | Use when |
|---|---|
| [`codeceptjs-fundamentals`](./codeceptjs-fundamentals/SKILL.md) | **Always run first.** Compact primer on CodeceptJS internals (helpers, `I`, DI container, `inject()`, custom helpers, plugins as hook listeners, the `await` rule), then reads the project's config and reports the active helper, plugins, env switching, and page objects. |
| [`codeceptjs-auth`](./codeceptjs-auth/SKILL.md) | A test needs login or different user roles. Configures the `auth` plugin from the real login page HTML, with env-var credentials and `secret()`-wrapped passwords. |
| [`codeceptjs-exploration`](./codeceptjs-exploration/SKILL.md) | Need to look at a page — read ARIA, inspect candidate elements, pick a stable locator. Covers the WebElement API and the broad-XPath candidate-discovery technique. Invoked by writing, debugging, and refactoring. |
| [`codeceptjs-run-analysis`](./codeceptjs-run-analysis/SKILL.md) | After `codeceptjs run` — read `output/trace_*/` artifacts via bash tools to verify a fix, cluster errors across many failures, or diagnose flakiness across reruns. Invoked by writing, debugging, refactoring, and ci-fix. |
| [`writing-codeceptjs-tests`](./writing-codeceptjs-tests/SKILL.md) | Authoring or extending tests. Drives the live page through MCP, learns locators from ARIA, builds Scenarios incrementally, commits the verified sequence. |
| [`debugging-codeceptjs-tests`](./debugging-codeceptjs-tests/SKILL.md) | A test is failing or flaky. Two modes — interactive via the MCP server, or non-interactive via `--debug` (escalating to `DEBUG="codeceptjs:*"` for framework-internal issues). |
| [`refactoring-codeceptjs-tests`](./refactoring-codeceptjs-tests/SKILL.md) | Cleaning up an existing suite — extracting page objects, taming long locators, moving raw JS into custom helpers. Targeted (one file) or global. Proposes before applying. |
| [`ci-fix-tests`](./ci-fix-tests/SKILL.md) | Non-interactive CI mode: a run failed; auto-attempt safe fixes (locator drift, missing waits), rerun only the failing scenarios, roll back if no improvement, always write `output/ci-fix.md`. |

`codeceptjs-fundamentals`, `codeceptjs-exploration`, `codeceptjs-run-analysis`, and `codeceptjs-auth` are utility skills referenced from the action skills (writing, debugging, refactoring, ci-fix). The action skills are what users typically invoke; the utility skills get pulled in as needed.

## Installing

The `SKILL.md` format is the [Agent Skills open standard](https://agentskills.io), which Claude Code already supports natively. For Cursor and Codex, a tiny extension or filename change is enough — the markdown body and frontmatter carry over.

All install commands below assume you are inside the directory that contains the skill folders — either a clone of this repo, or `skills/` inside a CodeceptJS project that vendored a copy. To get the standalone version:

```bash
git clone https://github.com/codeceptjs/skills.git
cd skills
```

### Claude Code

Skills live in one of three locations (project, personal, plugin) — see [Claude's skills docs](https://code.claude.com/docs/en/skills). Each skill is a directory with `SKILL.md`. Claude auto-loads a skill when the description matches the user's request, or you can invoke it directly with `/skill-name`.

**Project-local** (recommended — version-control alongside your tests):

```bash
mkdir -p .claude/skills
ln -s "$PWD"/*/ .claude/skills/   # or `cp -r` if symlinks are awkward
```

**Personal** (available across all your projects):

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD"/*/ ~/.claude/skills/
```

Claude watches these directories for changes — adding or editing a skill takes effect within the current session.

### Cursor

Cursor reads project rules from `.cursor/rules/*.mdc`. The frontmatter format is compatible (`description` is recognised), so the only thing that needs to change is the file extension and location.

```bash
mkdir -p .cursor/rules
for d in */; do
  name=$(basename "$d")
  cp "$d/SKILL.md" ".cursor/rules/$name.mdc"
done
```

For each rule, Cursor's Agent reads `description` to decide when to apply. To make a skill apply only when working in test files, edit the rule's frontmatter and add a `globs` field (e.g. `globs: ["tests/**/*.js", "tests/**/*.ts"]`); to make it always-on, add `alwaysApply: true`.

User-level rules (across all your Cursor projects) are managed through `Cursor Settings > Rules` — see [Cursor's rules docs](https://cursor.com/docs/context/rules).

### Codex CLI / `AGENTS.md`

Codex (and several other tools — GitHub Copilot, Aider, Cline, etc.) reads [`AGENTS.md`](https://agents.md), an open Markdown format that lives at the repo root. It has no native per-skill discovery, so the practical approach is to generate one `AGENTS.md` from all the skills.

**Option A — concatenate everything** (every skill always in context, simplest):

```bash
{
  echo "# CodeceptJS Agent Instructions"
  echo
  echo "AI skills for working with CodeceptJS 4 in this project. The full content of each is included below."
  echo
  for d in */; do
    cat "$d/SKILL.md"
    echo
  done
} > AGENTS.md
```

**Option B — pointer index** (lighter context, agent reads each `SKILL.md` on demand):

```bash
{
  echo "# CodeceptJS Agent Instructions"
  echo
  echo "AI skills for working with CodeceptJS 4. Open the relevant \`SKILL.md\` for your task:"
  echo
  for d in */; do
    name=$(basename "$d")
    desc=$(awk '/^description:/{sub(/^description: */, ""); print; exit}' "$d/SKILL.md")
    echo "- \`$name/SKILL.md\` — $desc"
  done
} > AGENTS.md
```

Pick A if you want every skill loaded for every task; pick B if your agent is happy following file references. Both work; B keeps the upfront context small.

`AGENTS.md` is also recognised by Claude Code and Cursor as a fallback when their native systems aren't set up — so even if a contributor hasn't run the install steps above, anything in `AGENTS.md` will still reach the agent.

### Other tools

The skill format is plain Markdown with YAML frontmatter, so any tool that reads project instructions can use them. Common patterns:

- **GitHub Copilot Workspace** — reads `AGENTS.md` natively.
- **Aider** — reads `CONVENTIONS.md`; rename or symlink an `AGENTS.md` into place.
- **Continue.dev** — point at this directory in `.continue/config.json`.

## Format

```
<skill-name>/
└── SKILL.md
```

Each `SKILL.md` starts with frontmatter and a body:

```yaml
---
name: <skill-name>
description: One sentence describing what the skill does and when it should trigger.
---

# Title

Skill body — workflows, rules, decision trees. Reference docs at
`node_modules/codeceptjs/docs/...` rather than duplicating them.
```

`description` is the field every supported tool relies on for triggering. Front-load the key use case in the description; tool budgets typically truncate descriptions around 1–1.5 KB.

Conventions used by skills in this repo:

- Sub-150 lines per skill; concept-driven prose over code dumps.
- Prefer pointers to `node_modules/codeceptjs/docs/<file>.md` over reproducing content.
- Cross-reference other skills by name (`see codeceptjs-exploration`) — let the agent navigate.
- No `await` on plain action steps in any code snippet (matches the framework's `await` rule).
- Credentials and secrets always come from env vars and pass through `secret()`.

## Contributing

Before adding a new skill:

1. Read every existing `SKILL.md` — they share a tone, a structure, and an opinion about brevity.
2. Check whether your idea is already covered by one of them. If you'd be duplicating triggers, extend the existing skill instead.
3. Make sure the workflow has a real goal (writing, fixing, refactoring) — toolkits without a goal are fine but should be split into "foundations" + "use cases" rather than a numbered workflow (see `codeceptjs-run-analysis` for the pattern).

When you add or rename a skill, update the table at the top of this README.
