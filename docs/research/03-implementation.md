# Implementation with Agentic Coding Tools

Research date: April 2026. Focus: quality-focused, senior-engineer workflows — not vibe coding.

---

## 1. Task Decomposition Strategies

The consensus across top practitioners is **decompose by capability, not by layer**. A "models commit" is never atomic — nobody can do anything with a model that has no controller or route. Each commit delivers one capability: something a user or system can do after this commit that they couldn't before.

### Scale-Based Sizing

Teams converge on sizing tasks relative to context window capacity and file scope:

| Scale | Files | Workflow |
|-------|-------|----------|
| Small | 1-2 | Plan → Implement → QA |
| Medium | 3-5 | Requirements → Design → Implement → QA |
| Large | 6+ | PRD → Architecture Decision → Design → Implement → QA |

Each subtask should complete within a single context window to avoid performance degradation from automatic compaction. At ~60% context utilization, output quality degrades silently — well before hitting hard limits.

Sources:
- [Zero Context Exhaustion: Production-Ready AI Coding Teams](https://dev.to/shinpr/zero-context-exhaustion-building-production-ready-ai-coding-teams-with-claude-code-sub-agents-31b) — processed 770K tokens across 8 tasks with zero exhaustion using independent context windows per agent
- [Claude Code Production: 40% Productivity Increase](https://dev.to/dzianiskarviha/integrating-claude-code-into-production-workflows-lbn) — nested CLAUDE.md files, subtasks sized to fit 200K context window
- [Context Window Management: 50 Sessions](https://blakecrosley.com/blog/context-window-management) — quality degrades at ~60% utilization

### The One-Sentence Litmus Test

> "Can't articulate the task in ONE sentence with ONE measurable outcome? Split it."

Atomic sessions cost less and produce cleaner code. In a direct comparison on a real bug fix, the atomic approach cost 59% less ($0.77 vs $1.82), produced ~90 lines across 2 files (vs 340+ lines across 6), and shipped without code review debate — while the bundled approach triggered multiple design questions.

Source: [Claude Code: Atomic Task Breakdown Test](https://medium.com/@levi_stringer/claude-code-one-shot-or-slow-down-bcb6283990d0)

### Spec-Driven Development (SDD)

GitHub's open-source [Spec Kit](https://github.com/github/spec-kit) (80K+ stars as of Feb 2026) formalizes a four-phase workflow:

1. **Specify** — Capture user journeys, not technical implementation. Focus on WHAT and WHY.
2. **Plan** — Technical architecture respecting organizational constraints.
3. **Tasks** — Decompose into small, reviewable chunks implementable in isolation.
4. **Implement** — Agent executes tasks sequentially or in parallel; developer reviews focused changes.

Key design: specs use `[NEEDS CLARIFICATION]` tags to force explicit identification of ambiguities rather than plausible guesses. Anti-speculation rules require every feature to trace to concrete user stories.

The methodology compresses traditional ~12-hour specification processes into ~15 minutes across three commands (`/speckit.specify`, `/speckit.plan`, `/speckit.tasks`).

Sources:
- [GitHub Blog: Spec-Driven Development](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [Spec Kit spec-driven.md](https://github.com/github/spec-kit/blob/main/spec-driven.md)
- [Martin Fowler: Understanding SDD Tools](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)

### Real Example: 14 Tasks, 14 Commits, 45 Minutes

An SQLite-to-IndexedDB migration used spec-driven decomposition with dependency-aware parallel execution:

```json
{
  "id": "task-1",
  "subject": "Create idb-helpers.ts",
  "status": "pending",
  "blocks": ["task-3", "task-4"],
  "blockedBy": ["task-0"]
}
```

Tasks with no blockers ran concurrently in "waves." 14 tasks produced 14 atomic commits across 15+ files at 71% context utilization. The orchestrator stayed lean — only handling task coordination and commits.

Source: [Spec-Driven Development in Action](https://alexop.dev/posts/spec-driven-development-claude-code-in-action/)

---

## 2. Prompt Engineering for Code Generation

### CLAUDE.md: Less Is More

The official guidance is ruthless pruning. For each line, ask: *"Would removing this cause Claude to make mistakes?"* If not, cut it. Bloated CLAUDE.md files cause Claude to ignore actual instructions.

**Include:**
- Bash commands Claude can't guess
- Code style rules that differ from defaults
- Testing instructions and preferred test runners
- Repository etiquette (branch naming, PR conventions)
- Architectural decisions specific to your project
- Common gotchas or non-obvious behaviors

**Exclude:**
- Anything Claude can figure out by reading code
- Standard language conventions Claude already knows
- Detailed API documentation (link to docs instead)
- File-by-file descriptions of the codebase
- Self-evident practices like "write clean code"

Use emphasis (e.g., "IMPORTANT" or "YOU MUST") for critical rules. Check CLAUDE.md into git so your team contributes. Treat it like code — review when things go wrong, prune regularly.

Source: [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)

### Constraint-Based vs Example-Based Prompting

The most effective approach combines both:

- **Constraints** define boundaries: "No mocks. No service objects. Fat models, skinny controllers."
- **Examples** anchor patterns: "Look at HotDogWidget.php — follow this pattern for the new calendar widget."

For rules files (.cursorrules, CLAUDE.md), the Cursor team recommends: "Reference files rather than copying their contents to prevent staleness." And: "Start simple. Add rules only when you notice the agent making the same mistake repeatedly."

Sources:
- [Cursor: Best Practices for Coding with Agents](https://cursor.com/blog/agent-best-practices)
- [Martin Fowler: Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

### Skills over Bloated CLAUDE.md

For domain knowledge or workflows relevant only sometimes, use skills (`.claude/skills/`) instead of CLAUDE.md. Claude loads them on-demand without bloating every conversation. Skills are folders that use `references/`, `scripts/`, and `examples/` subdirectories for progressive disclosure.

The skill format has become a universal standard — the same SKILL.md files work across Claude Code, Cursor, Gemini CLI, Codex CLI, and Antigravity IDE as of March 2026.

Sources:
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/best-practices)
- [VoltAgent: Awesome Agent Skills](https://github.com/VoltAgent/awesome-agent-skills)

### Give Claude Verification, Not Just Instructions

> "Include tests, screenshots, or expected outputs so Claude can check itself. This is the single highest-leverage thing you can do."

The official best practices doc identifies this as the #1 recommendation. Verification can be a test suite, a linter, a Bash command that checks output, or screenshot comparison via the Chrome extension.

Source: [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)

---

## 3. Multi-Agent Implementation Patterns

### Pattern A: Architect / Editor Split (Aider)

Aider's architect mode separates reasoning from editing:

1. **Architect model** (e.g., Claude Opus) describes how to solve the problem
2. **Editor model** (e.g., Claude Sonnet, Deepseek) turns the proposal into specific file edits

This achieved 85% on Aider's code editing benchmark (o1-preview + Deepseek/o1-mini), significantly outperforming any single model. The key insight: each model focuses on its strength rather than doing both tasks simultaneously.

Source: [Aider: Separating Code Reasoning and Editing](https://aider.chat/2024/09/26/architect.html)

### Pattern B: Writer / Reviewer with Fresh Context

Claude Code's official docs recommend a two-session pattern:

| Session A (Writer) | Session B (Reviewer) |
|---|---|
| Implement rate limiter | — |
| — | Review `rateLimiter.ts` for edge cases, race conditions |
| Address review feedback | — |

A model reviewing its own code in the same session carries all its assumptions forward. A fresh context eliminates this bias. Claude Code Review (launched March 2026) formalizes this — it dispatches multiple specialized agents and includes a falsification step that attempts to disprove each finding before posting. Internal numbers: findings jumped from 16% to 54% of PRs with <1% incorrect findings.

Sources:
- [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic launches code review tool](https://techcrunch.com/2026/03/09/anthropic-launches-code-review-tool-to-check-flood-of-ai-generated-code/)

### Pattern C: Multi-Provider Review (Korokithakis)

Stavros Korokithakis runs a three-tier pipeline:
- **Architect**: Claude Opus 4.6 for planning
- **Developer**: Claude Sonnet 4.6 for implementation
- **Reviewers**: OpenAI Codex, Google Gemini, and Anthropic Opus in parallel

The critical design principle: "Single-model review loops suffer from self-agreement bias: a model will rarely robustly critique its own output." Using competing vendors' models is a *functional requirement*, not preference.

He sustained codebases reaching tens of thousands of lines over weeks, though stability depends on domain familiarity — established expertise maintained better than unfamiliar territories. Key lesson: "LLMs sharpen what a developer already knows rather than substitute for knowledge they don't have."

Source: [Agent Wars: Multi-Agent LLM Workflows](https://agent-wars.com/news/2026-03-16-how-one-developer-uses-multi-agent-llm-workflows-architect-developer-reviewers)

### Pattern D: Parallel Sub-Agents with Orchestrator

Claude Code supports three parallelization approaches:

1. **Sub-agents** — Run in own context, report back to caller. Best for focused tasks where only the result matters. Lower token cost.
2. **Agent Teams** (experimental) — Shared task list, direct inter-agent messaging. Best for complex work requiring discussion. Higher token cost.
3. **Git Worktrees** — Manual parallel sessions in isolated directories. Full independence, no coordination overhead.

Sub-agent dispatch rules — ALL conditions must be met:
- 3+ unrelated tasks or independent domains
- No shared state between tasks
- Clear file boundaries with no overlap

Common mistake: "Sub-agent failures aren't execution failures — they're invocation failures." Give comprehensive context upfront since sub-agents can't ask clarifying questions.

Sources:
- [Claude Code: Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Sub-Agent Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices)

### Pattern E: Specialized Agent Teams (9 Agents)

A production-ready architecture uses nine specialized agents:

| Agent | Role | Context |
|-------|------|---------|
| requirement-analyzer | Scope assessment | 30K tokens |
| technical-designer | Architecture docs | 60K tokens |
| task-decomposer | Break down work | 20K tokens |
| task-executor | TDD implementation | 5-60K per task |
| quality-fixer | Lint/type/test | 90K per task |
| rule-advisor | Dynamic rule selection | 15K tokens |

The `rule-advisor` implements meta-cognition: it intercepts work requests, analyzes task essence, and selects only relevant rules — preventing the "too helpful" bias where LLMs rush to answer without proper grounding.

Source: [Zero Context Exhaustion](https://dev.to/shinpr/zero-context-exhaustion-building-production-ready-ai-coding-teams-with-claude-code-sub-agents-31b)

---

## 4. Git Workflow Integration

### Worktrees for Parallel Agents

Git worktrees have become the standard isolation mechanism. Each worktree gets its own branch and working directory while sharing repository history.

```bash
claude --worktree feature-auth  # Creates worktree + starts Claude in it
```

incident.io runs "seven ongoing conversations at once, each evolving independently" using a custom bash function (`w`) that automates worktree creation:

```bash
w myproject new-feature claude    # Create worktree + launch Claude
w myproject new-feature git commit -m "fix: the thing"  # Remote command
```

Sources:
- [incident.io: Shipping Faster with Claude Code and Git Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees)
- [Claude Code Common Workflows](https://code.claude.com/docs/en/common-workflows)

### Atomic Commits with Agents

The production workflow enforces one commit per completed task. Pre-commit hooks provide automated backpressure — if validation fails, the agent sees errors and self-corrects.

Quality gates via Claude Code hooks:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git commit*)",
        "command": "npm run lint && npm run typecheck && npm test"
      }
    ]
  }
}
```

If the hook exits non-zero, the commit is blocked and Claude sees the error output. This is deterministic — unlike CLAUDE.md instructions which are advisory, hooks guarantee the action happens.

Sources:
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Hooks Tutorial](https://blakecrosley.com/blog/claude-code-hooks-tutorial)

### Commit Patterns in Practice

A production monorepo team tracked these metrics:

| Metric | Pre-Claude | With Claude |
|--------|-----------|-------------|
| Avg commits/week | ~28 | ~50 |
| Source LOC/week | ~1,822 | ~3,473 |
| Test LOC/week | ~505 | ~2,043 |

Commits became more atomic — previously combined multiple changes into single commits. This improved CI/CD and code review clarity.

Source: [Claude Code in Production: 40% Productivity Increase](https://dev.to/dzianiskarviha/integrating-claude-code-into-production-workflows-lbn)

---

## 5. Context Management

### The Fundamental Constraint

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."
> — Anthropic official docs

### Practical Rules

1. **`/clear` between unrelated tasks** — Long sessions with irrelevant context reduce performance.
2. **Compact every 25-30 minutes** during intensive work, or after completing a distinct subtask.
3. **Delegate research to sub-agents** — They explore in separate context, report back summaries.
4. **Don't pre-load files** — Let tools find relevant files on demand. Pre-loading 8-10 files "for context" wastes tokens.
5. **Use line offsets** — Read specific functions, not entire files.
6. **After 2 failed corrections, `/clear` and restart** with a better prompt incorporating what you learned.

### Handoff Documents for Multi-Session Work

For tasks spanning sessions, create structured summaries capturing:
- Current status
- Modified files
- Architectural decisions
- Blockers
- Next steps

This reduces token costs for successor sessions needing full context.

### The Interview Pattern

For larger features, have Claude interview you first:

> "I want to build [brief description]. Interview me in detail using the AskUserQuestion tool. Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs. Don't ask obvious questions, dig into the hard parts I might not have considered."

Once the spec is complete, start a fresh session to execute it. The new session has clean context focused entirely on implementation.

Sources:
- [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Context Window Management: 50 Sessions](https://blakecrosley.com/blog/context-window-management)
- [Martin Fowler: Context Engineering](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

### Context Engineering > Prompt Engineering

The industry has shifted terminology from "prompt engineering" to "context engineering" — curating the full information ecosystem, not just the task instruction. Five core strategies:

1. **Selection** — Choose what context to include
2. **Compression** — Minimize tokens per unit of information
3. **Ordering** — Place critical information at start and end (models attend to these best)
4. **Isolation** — Separate concerns into independent context windows
5. **Format** — Structure information for LLM consumption

The "lost-in-the-middle" problem is real: research from Stanford and UC Berkeley found model correctness drops around 32K tokens even for models claiming much larger windows. Models focus on the beginning and end.

Sources:
- [Faros: Context Engineering for Developers](https://www.faros.ai/blog/context-engineering-for-developers)
- [Martin Fowler: Context Engineering](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

---

## 6. Real-World Configurations and Examples

### Anthropic's 2026 Trends Report: Key Data Points

- Developers integrate AI into 60% of their work
- Active oversight maintained on 80-100% of delegated tasks
- Only senior+ engineers successfully use parallel agents so far
- Task horizons expanding from minutes to days/weeks
- Organizations shifting to "delegate, review, and own" operating model

Source: [Anthropic 2026 Agentic Coding Trends](https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report/)

### Production CLAUDE.md Structure (Real Team)

From a monorepo with 40% productivity gains:

```
project-root/
  CLAUDE.md              # Project-wide conventions
  backend/
    CLAUDE.md            # Backend-specific patterns
  frontend/
    CLAUDE.md            # Frontend-specific patterns
  .claude/
    agents/              # Specialized sub-agents
      backend-code-reviewer.md
      frontend-code-reviewer.md
    skills/              # On-demand workflows
      fast-workflow/
      full-workflow/
    commands/            # Reusable slash commands
```

Each subtask generates an implementation plan, to-do checklist, and implementation overview before coding starts.

Source: [Claude Code in Production](https://dev.to/dzianiskarviha/integrating-claude-code-into-production-workflows-lbn)

### Cursor Rules File Pattern

Cursor's official recommendation for `.cursor/rules/`:

- Store persistent instructions as markdown files
- Include essential commands, code patterns, and pointers to canonical examples
- Define reusable commands in `.cursor/commands/` (stored in git)
- Standard commands: `/pr`, `/fix-issue [number]`, `/review`, `/update-deps`

Source: [Cursor: Agent Best Practices](https://cursor.com/blog/agent-best-practices)

### TDD Pattern with Agents

From Cursor's best practices:

1. Request tests first based on input/output pairs (explicitly state TDD)
2. Run tests — confirm they fail without implementation
3. Commit tests
4. Have agent write code passing tests (no test modifications)
5. Commit implementation

> "Agents perform best when they have a clear target to iterate against."

Source: [Cursor: Agent Best Practices](https://cursor.com/blog/agent-best-practices)

### Quality Gate Hooks (Claude Code)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "npx eslint --fix ${file}"
      }
    ],
    "TeammateIdle": [
      {
        "command": "npm run lint && npm run typecheck",
        "exitCode2": "Tests still failing — keep working"
      }
    ]
  }
}
```

Source: [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks)

---

## 7. Key Patterns for Skill Design

Synthesizing across all research, the patterns that matter most for a quality-focused engineering toolkit:

### The Explore-Plan-Implement-Commit Cycle

Every non-trivial task should follow this sequence. Plan Mode for exploration, spec document as persistent artifact, fresh context for implementation, verification before commit. This is the workflow Anthropic recommends internally.

### Capability-Ordered Decomposition

Decompose features by capability, not by layer. Each commit delivers one thing a user/system can do that they couldn't before. Model methods arrive in the commit where they're first used — don't front-load untested code.

### Spec as Recovery Point

Written specifications survive session restarts and coordinate across sessions. Unlike conversation memory, specs are persistent, reviewable, and shareable. The spec is the source of truth, not the conversation.

### Independent Context Per Task

Every sub-agent gets fresh context for its specific task. The orchestrator stays lean — handling only coordination and commits. This prevents the "accumulated cruft" problem that degrades quality in long sessions.

### Deterministic Quality Gates

Hooks > instructions. Pre-commit hooks that run lint, typecheck, and tests are non-negotiable. Advisory instructions in CLAUDE.md get ignored under context pressure; hooks are deterministic.

### Cross-Model Review

A model reviewing its own output in the same context carries forward all its assumptions. Fresh context (separate session) or different model (cross-provider review) catches what the author missed. This is not optional for quality-focused work.
