# Skills

Agentic development skills for defining, designing, implementing, and reviewing features. Built on the [Agent Skills](https://agentskills.io) open standard — works with Claude Code, Codex, and other compatible tools.

## Installation

### Claude Code

Symlink each skill into `~/.claude/skills/`:

```bash
ln -sf /path/to/this/repo/define ~/.claude/skills/define
ln -sf /path/to/this/repo/design ~/.claude/skills/design
ln -sf /path/to/this/repo/execute ~/.claude/skills/execute
ln -sf /path/to/this/repo/review ~/.claude/skills/review
ln -sf /path/to/this/repo/forge ~/.claude/skills/forge
```

### OpenAI Codex

Symlink each skill into `~/.agents/skills/` (or place in `.agents/skills/` at your repo root):

```bash
ln -sf /path/to/this/repo/define ~/.agents/skills/define
ln -sf /path/to/this/repo/design ~/.agents/skills/design
ln -sf /path/to/this/repo/execute ~/.agents/skills/execute
ln -sf /path/to/this/repo/review ~/.agents/skills/review
ln -sf /path/to/this/repo/forge ~/.agents/skills/forge
```

Note: Codex support is untested. The skills follow the Agent Skills standard which Codex supports, but sub-agent spawning and checklist loading may behave differently. The checklists reference "CLAUDE.md conventions" — adapt to "AGENTS.md conventions" for Codex.

### Other Tools

Any tool supporting the [Agent Skills standard](https://agentskills.io) (Cursor, Gemini CLI, Copilot, Kiro, Junie, and others) can load these skills. Check your tool's documentation for the skill installation path.

The shared review checklists in `checklists/` are loaded by the skills at runtime.

## The Pipeline

```
/define  →  /design  →  /execute  →  /review
```

**`/define`** — Define WHAT to build and WHY. Outputs a GitHub issue with problem statement, behaviors, decisions, constraints, and edge cases. Challenges the premise before accepting requirements.

**`/design`** — Design HOW to build it. Reads the issue from /define, explores the codebase, proposes competing approaches, produces a capability-ordered implementation plan appended to the same issue.

**`/execute`** — Implement commit by commit. Reads the plan from /design, implements each capability atomically with independent code review per commit, manages context across long sessions.

**`/review`** — Interactive code review with history surgery. Walks through commits, cross-references the original spec to catch completion bias, rewrites history directly on private branches.

**`/forge`** — Builds and tests skills. A Ralph-style loop that iterates (draft → test → improve) until the skill passes adversarial testing and goal-level evaluation.

## How It Works

Each skill in the pipeline produces an artifact that feeds the next:
- `/define` produces a **GitHub issue** (the spec)
- `/design` appends an **implementation plan** to the same issue
- `/execute` produces **commits** on a feature branch, with deviations as issue comments
- `/review` produces a **clean commit history** verified against the original spec

Pipeline routing is a human decision. Not every task needs every step — a typo fix skips straight to implementation, a small change might skip /design.

## Checklists

The `checklists/` directory contains review criteria loaded by sub-agents during /define, /design, and /execute:

- `spec-review.md` — For reviewing product specs (used by /define)
- `convention-review.md` — For reviewing code against project conventions
- `adversarial-review.md` — For adversarial challenge of any artifact

## Research

The skills were built from extensive research into agentic coding best practices. See [docs/research/README.md](docs/research/README.md) for the full backstory, or start with [docs/research/00-synthesis.md](docs/research/00-synthesis.md) for the unified spec.
