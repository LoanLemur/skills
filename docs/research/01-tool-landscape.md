# Agentic Coding Tools: Landscape Survey (April 2026)

Research compiled for downstream agents building a unified skills/configuration spec.

---

## Table of Contents

1. [Claude Code (Anthropic)](#1-claude-code-anthropic)
2. [OpenAI Codex](#2-openai-codex)
3. [Cursor](#3-cursor)
4. [Windsurf (Cognition/Codeium)](#4-windsurf-cognitioncodeium)
5. [Google Antigravity](#5-google-antigravity)
6. [Kiro (AWS/Amazon)](#6-kiro-awsamazon)
7. [GitHub Copilot (IDE + CLI)](#7-github-copilot-ide--cli)
8. [Aider](#8-aider)
9. [Cline](#9-cline)
10. [Devin (Cognition)](#10-devin-cognition)
11. [Factory](#11-factory)
12. [Augment Code](#12-augment-code)
13. [Comparison Matrix](#13-comparison-matrix)
14. [Assessment: What Matters for Quality-Focused Teams](#14-assessment-what-matters-for-quality-focused-teams)

---

## 1. Claude Code (Anthropic)

**Surface:** Terminal CLI (primary), SDK for embedding  
**Model:** Claude Opus 4.6, Sonnet 4.6  
**Open source:** Partial (hooks/skills spec is open; core is proprietary)  
**GitHub stars:** 82K+ (as of March 2026)

### Core Philosophy

Claude Code is a terminal-native agentic coding tool. Its architecture treats the developer as the orchestrator and the agent as a capable subordinate that reads, writes, and executes within the developer's own environment. The design prioritizes composability with unix tools, explicit permission boundaries, and a layered configuration system that separates deterministic behavior (always runs) from probabilistic behavior (Claude decides).

### Feature Development Pipeline

Claude Code does not impose a fixed pipeline. Instead, it provides building blocks:

- **Spec/design:** Users can create custom skills (e.g., `/define`, `/design`) that template spec-driven workflows. The CLAUDE.md file encodes project-level standards that apply to all work.
- **Implementation:** The main agent loop reads files, proposes edits, runs terminal commands, and iterates. It uses an "explore" subagent for codebase analysis before making changes.
- **Review:** PostToolUse hooks can run linters, formatters, and tests after every file write. Custom review subagents can be configured.

### Configuration Mechanisms

Claude Code has four layers, split by determinism:

| Layer | Deterministic? | Scope | Purpose |
|-------|---------------|-------|---------|
| **CLAUDE.md** | Yes (always loaded) | Per-repo, per-user, per-directory | Project conventions, coding standards, architecture rules |
| **Hooks** | Yes (guaranteed to fire) | Per-project or per-user (settings.json) | Lifecycle callbacks: 24 events including PreToolUse, PostToolUse, SessionStart, Stop, SubagentStart, PermissionRequest, FileChanged, etc. Three handler types: command (bash), prompt (inject text), agent (spawn subagent) |
| **Skills** | No (Claude decides) | Per-project (.claude/skills/) or per-user | Reusable workflows with SKILL.md frontmatter + markdown body. Follows the open Agent Skills standard. Can be invoked manually via slash commands or automatically by Claude's judgment |
| **Subagents** | No (Claude decides) | Per-project or per-user | Specialized agents with isolated context windows, scoped tool access, custom system prompts. Cannot spawn their own subagents |

**Rules files:** `.claude/rules/*.md` for glob-matched contextual rules (e.g., rules that apply only to `*.test.ts` files).

### Sub-Agent / Multi-Agent Capabilities

- **Subagents** run in isolated context windows. The parent passes a prompt string; the subagent returns its final message. No shared memory beyond what's explicitly passed.
- **Built-in subagent:** "Explore" -- read-only, optimized for codebase search with configurable thoroughness (quick/medium/very thorough).
- **Agent Teams** (separate from subagents): coordinate across separate sessions for parallel work. Subagents are within a single session; teams are cross-session.
- Subagents inherit the parent's permission policy but can have tool access further restricted. They cannot exceed parent permissions or spawn their own subagents.
- Subagents can target cheaper/faster models (e.g., Haiku) for cost control.

### Strengths

- Deepest configuration system in the market (4 layers, 24 hook events, 3 handler types)
- Terminal-native composes with git, unix tools, CI/CD pipelines
- Explicit permission model with deterministic hooks for safety
- Strong community ecosystem (awesome-claude-code, 82K+ GitHub stars)
- CLAUDE.md is the de facto standard for repo-level AI instructions
- SWE-bench Verified score of 80.8% (top as of early 2026)

### Weaknesses

- No built-in visual/GUI feedback (terminal only)
- Steep learning curve for full configuration (hooks + skills + subagents + CLAUDE.md)
- Token-intensive for large codebases without careful context management
- Probabilistic skill/subagent invocation means you can't guarantee Claude uses them

### Community & Maturity

Largest open-source community among agentic coding tools. 82K+ GitHub stars. Extensive third-party skill libraries, hooks collections, and integration guides. Anthropic publishes an annual "Agentic Coding Trends Report."

**Sources:**
- [Claude Code Skills Docs](https://code.claude.com/docs/en/skills)
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Subagents Docs](https://code.claude.com/docs/en/sub-agents)
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
- [Claude Code Architecture Explained](https://dev.to/brooks_wilson_36fbefbbae4/claude-code-architecture-explained-agent-loop-tool-system-and-permission-model-rust-rewrite-41b2)

---

## 2. OpenAI Codex

**Surface:** Cloud agent (primary), CLI (open-source terminal tool)  
**Model:** GPT-5-Codex (GPT-5 optimized for coding), codex-1 (o3-based)  
**Open source:** CLI is open-source; cloud agent is proprietary

### Core Philosophy

Codex takes a dual approach: a cloud-based agent for autonomous parallel task execution, and an open-source CLI for local interactive work. The cloud agent emphasizes safety through sandboxing -- every task runs in its own isolated environment with network access disabled by default. The philosophy is "let the agent work independently in a safe box, then review the output."

### Feature Development Pipeline

- **Cloud agent:** Accepts tasks via ChatGPT or API. Each task gets a sandbox preloaded with the repository. The agent plans, implements, tests, and proposes a PR. Multiple tasks run in parallel.
- **CLI:** Interactive terminal agent with human-in-the-loop approval. Supports `suggest`, `auto-edit`, and `full-auto` approval modes for graduated autonomy.
- **AGENTS.md:** Repository-level configuration file (analogous to CLAUDE.md) that tells Codex about project structure, testing commands, and coding standards.

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **AGENTS.md** | Repo-level instructions. Discovered hierarchically: AGENTS.override.md > AGENTS.md > TEAM_GUIDE.md > .agents.md |
| **config.toml** | User-level config (~/.codex/config.toml) for model selection, approval mode, sandbox settings |
| **Agent Skills** | Codex supports the Agent Skills open standard (same as Claude Code) for reusable workflows |
| **MCP support** | Connect to external tools and context via Model Context Protocol |

### Sub-Agent Capabilities

- Cloud Codex runs tasks as independent sandboxed agents (not sub-agents of a parent, but parallel independent workers).
- The CLI can be orchestrated via the OpenAI Agents SDK for programmatic multi-agent workflows.
- No built-in sub-agent delegation like Claude Code's explore agent.

### Strengths

- Cloud sandbox model is the strongest security story (network-disabled, isolated environments)
- Parallel task execution in the cloud is genuinely useful for batch work (migrations, test coverage)
- Open-source CLI with good community
- AGENTS.md is well-documented and hierarchical

### Weaknesses

- Cloud agent latency: tasks take minutes, not seconds
- Sandbox restrictions limit tools that need network access
- Less mature configuration system than Claude Code (no hooks, no deterministic lifecycle events)
- CLI is simpler than Claude Code's terminal experience

### Community & Maturity

Strong backing from OpenAI. The CLI has a healthy open-source community. Cloud Codex is integrated into ChatGPT Pro/Team/Enterprise, giving it broad reach. Terminal-Bench and SWE-bench scores are competitive but generally trail Claude Code.

**Sources:**
- [Introducing Codex](https://openai.com/index/introducing-codex/)
- [AGENTS.md Guide](https://developers.openai.com/codex/guides/agents-md)
- [Codex CLI GitHub](https://github.com/openai/codex)
- [Codex Changelog](https://developers.openai.com/codex/changelog)
- [Codex with Agents SDK](https://developers.openai.com/codex/guides/agents-sdk)

---

## 3. Cursor

**Surface:** IDE (VS Code fork)  
**Model:** Composer (proprietary, 4x faster than competitors), plus Claude, GPT, Gemini  
**Pricing:** Free tier, Pro ($20/mo), Business ($40/mo), Enterprise

### Core Philosophy

Cursor is the dominant IDE-first agentic coding tool. It started as an AI-enhanced editor and has evolved into a multi-agent orchestration platform. The core bet is that developers want AI deeply integrated into their visual editing workflow, not in a separate terminal. Cursor 3.0 (April 2026) marks the shift from "AI-assisted editor" to "agent orchestration platform."

### Feature Development Pipeline

- **Composer/Agent mode:** Describe what you want; Cursor plans multi-step edits across files, runs terminal commands, and iterates. Agents can build, test, and demo features end-to-end.
- **Background Agents:** Delegate tasks to cloud sandboxes that work asynchronously. Results appear when ready.
- **Automations:** Always-on agents triggered by events (e.g., new PR, file change) with custom instructions and MCP integrations.
- **Design Mode (Cursor 3):** Annotate UI directly to guide implementation.

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **.cursorrules** | Repo-level instructions for code style, conventions, architecture. Cursor's equivalent of CLAUDE.md |
| **Agent Skills** | Reusable markdown-based workflows (adopted Jan 2026) |
| **MCP support** | Connect external tools |
| **Automations** | Event-triggered agent workflows |

### Sub-Agent / Multi-Agent Capabilities

- **Cursor 3.0 Agents Window:** Run up to 8 agents in parallel across repos and environments (local, worktrees, cloud, remote SSH).
- **Self-hosted cloud agents:** Enterprise feature for running agents in customer infrastructure (up to 10 workers/user, 50/team).
- **Background delegation:** Prefix prompts with `&` to offload to cloud agents.
- **Worktree-based comparison:** Run multiple agents on the same task and compare outputs.

### Strengths

- Best-in-class IDE integration with visual feedback
- Composer model is fast (sub-30-second turns)
- Multi-agent parallel orchestration (Cursor 3.0)
- Largest market share among IDE-based tools
- Self-hosted option for enterprise security

### Weaknesses

- .cursorrules is less expressive than CLAUDE.md + hooks (no lifecycle events, command-only handlers)
- Proprietary and closed-source
- VS Code lock-in (JetBrains support added March 2026 but less mature)
- Expensive at scale for teams

### Community & Maturity

Massive adoption. Cursor is the most-used agentic IDE as of 2026. Strong community around .cursorrules sharing and Composer workflows. Enterprise customers include major tech companies.

**Sources:**
- [Cursor Features](https://cursor.com/features)
- [Cursor 3.0 Changelog](https://cursor.com/changelog)
- [Self-hosted Cloud Agents](https://cursor.com/blog/self-hosted-cloud-agents)
- [Cursor 3 Guide](https://www.digitalapplied.com/blog/cursor-3-agents-window-design-mode-complete-guide)

---

## 4. Windsurf (Cognition/Codeium)

**Surface:** IDE (VS Code fork)  
**Model:** Multi-model (Claude, GPT, Gemini, custom)  
**Status:** Acquired by Cognition AI (Devin) in December 2025 for ~$250M

### Core Philosophy

Windsurf's core innovation is "flow awareness" -- Cascade (its agent) tracks everything you do (files edited, terminal commands, clipboard, conversation history) and uses this shared timeline to infer intent. The philosophy is that the best AI assistant is one that understands your context without you explaining it.

### Feature Development Pipeline

- **Cascade:** Multi-step agentic editing with deep repo context. Plans edits, calls tools, and coordinates across files from a single instruction.
- **Flows:** Reusable markdown-based workflows (terminal snippets and sequences saved as commands).
- **Agent Skills:** Adopted the Agent Skills standard as of January 2026.

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **Windsurf Rules** | Repo-level conventions (similar to .cursorrules) |
| **Agent Skills** | Reusable workflow definitions |
| **Flows** | Saved multi-step terminal/editing sequences |
| **MCP support** | External tool integration |

### Sub-Agent Capabilities

Limited compared to Claude Code or Cursor 3.0. Cascade operates as a single agent with flow awareness rather than spawning sub-agents. The Cognition acquisition may bring Devin's multi-agent architecture into Windsurf.

### Strengths

- Flow awareness is genuinely differentiated (context without prompting)
- Ranked #1 in LogRocket AI Dev Tool Power Rankings (Feb 2026)
- Good free tier
- Cognition acquisition brings Devin's autonomous capabilities

### Weaknesses

- Acquisition uncertainty: product direction unclear as Cognition integrates
- Less mature configuration system than Claude Code or Cursor
- Weaker multi-agent story than Cursor 3.0

### Community & Maturity

Large user base inherited from Codeium. Active changelog. The Cognition acquisition adds enterprise credibility but creates uncertainty about independent product direction.

**Sources:**
- [Windsurf Cascade](https://windsurf.com/cascade)
- [Windsurf Review 2026](https://vibecoding.app/blog/windsurf-review)
- [Windsurf Changelog](https://windsurf.com/changelog)

---

## 5. Google Antigravity

**Surface:** IDE (VS Code fork) with Manager View  
**Model:** Gemini 3.1 Pro, Gemini 3 Flash (also supports Claude Opus/Sonnet 4.6, GPT-OSS-120B)  
**Pricing:** Free in public preview with generous Gemini rate limits

### Core Philosophy

Antigravity is Google's "agent-first" IDE, announced November 2025 alongside Gemini 3. Unlike tools that added agents to an editor, Antigravity was designed from the ground up for multi-agent orchestration. Its distinguishing feature is the Manager View -- a mission control dashboard for dispatching, monitoring, and reviewing multiple parallel agents.

### Feature Development Pipeline

- **Editor View:** Standard AI-assisted coding with agent sidebar (similar to Cursor/Copilot).
- **Manager View:** Dispatch multiple agents to work in parallel across workspaces. Each agent produces "Artifacts" (task lists, implementation plans, screenshots, browser recordings) that can be reviewed and commented on.
- **Artifact System:** Agents generate verifiable deliverables. You can leave feedback directly on artifacts (like commenting on a doc), and agents incorporate feedback without stopping.

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **Project Rules** | Repo-level conventions |
| **Agent Skills** | Standard agent skills support |
| **MCP support** | External tool integration |
| **Gemini Code Assist integration** | Enterprise features via Google Cloud |

### Sub-Agent / Multi-Agent Capabilities

Strongest multi-agent orchestration UI in the market:
- Dispatch 5+ agents simultaneously on different tasks
- Real-time progress monitoring
- Artifact-based review (approve/reject changes before applying)
- Feedback loop: comment on artifacts mid-execution
- Cross-workspace agent coordination

### Strengths

- Best multi-agent orchestration UI (Manager View)
- Artifact system creates verifiable, reviewable deliverables
- Free with generous rate limits (strong onboarding story)
- Model-agnostic (supports Claude, GPT alongside Gemini)
- Google Cloud integration for enterprise

### Weaknesses

- Still in public preview (not GA)
- Gemini 3 model quality trails Claude Opus 4.6 on coding benchmarks
- Configuration system is less mature than Claude Code's
- Google's track record with developer tools creates trust concerns (product longevity)

### Community & Maturity

Early-stage but growing fast due to free access and Google's reach. Active developer blog and Codelabs tutorials. Community is smaller than Cursor or Claude Code.

**Sources:**
- [Google Developers Blog: Antigravity](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/)
- [Getting Started with Antigravity](https://codelabs.developers.google.com/getting-started-google-antigravity)
- [Antigravity Review 2026](https://leaveit2ai.com/ai-tools/code-development/antigravity)
- [Antigravity: The Agentic IDE](https://www.index.dev/blog/google-antigravity-agentic-ide)

---

## 6. Kiro (AWS/Amazon)

**Surface:** IDE (VS Code fork)  
**Model:** Claude (Anthropic) and Amazon Nova models  
**Status:** GA since November 2025; autonomous agent in preview

### Core Philosophy

Kiro's differentiator is **spec-driven development**. Where other tools start with "describe what you want and I'll code it," Kiro starts with "describe your requirements and I'll produce a spec, then code from the spec." This addresses the "vibe coding" problem where AI generates code without clear requirements, leading to drift and technical debt.

### Feature Development Pipeline

This is Kiro's strongest area -- it has an explicit pipeline:

1. **Requirements:** Developer describes requirements in natural language
2. **Spec generation:** Kiro produces user stories with acceptance criteria, a technical design document, and a task list
3. **Implementation:** Kiro works through tasks systematically, guided by the spec
4. **Autonomous agent (preview):** Can work for hours/days on tasks asynchronously, maintaining context across sessions

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **Steering Files** | Markdown files encoding project conventions, patterns, libraries, and standards. Persistent knowledge that applies to all interactions |
| **Agent Hooks** | Automated triggers on file/workspace events (save, create, delete). Execute predefined agent actions |
| **Specs** | Generated and versioned specification documents that guide implementation |

### Sub-Agent Capabilities

- **Autonomous agent (preview):** Can work independently for extended periods (hours/days), maintaining context across sessions and learning from feedback.
- No explicit sub-agent spawning model documented; Kiro operates more as a single persistent agent than a multi-agent system.

### Strengths

- Only tool with a genuine spec-first workflow (addresses the biggest gap in agentic coding)
- Steering files are well-designed for encoding conventions
- Agent hooks on file events are practical and intuitive
- AWS ecosystem integration
- Autonomous agent can work for days

### Weaknesses

- Spec generation quality varies (sometimes over-engineers simple tasks)
- Smaller community than Cursor/Claude Code
- AWS-centric ecosystem may limit model choice
- Autonomous agent still in preview

### Community & Maturity

Growing but smaller community. GA since November 2025. Strong AWS/enterprise backing. Active GitHub repo. Used in regulated industries (pharma, finance) where spec-driven development is valued.

**Sources:**
- [Kiro.dev](https://kiro.dev/)
- [Kiro: Spec-Driven Agentic AI IDE (InfoQ)](https://www.infoq.com/news/2025/08/aws-kiro-spec-driven-agent/)
- [Kiro GitHub](https://github.com/kirodotdev/Kiro)
- [Kiro Autonomous Agent (TechCrunch)](https://techcrunch.com/2025/12/02/amazon-previews-3-ai-agents-including-kiro-that-can-code-on-its-own-for-days/)

---

## 7. GitHub Copilot (IDE + CLI)

**Surface:** VS Code extension (primary), CLI (GA Feb 2026), GitHub.com integration  
**Model:** Multi-model (Claude Opus/Sonnet 4.6, GPT-5.3-Codex, Gemini 3 Pro)  
**Pricing:** Free tier, Pro ($10/mo), Business ($19/mo), Enterprise ($39/mo)

### Core Philosophy

Copilot has evolved from autocomplete to a full agentic platform across three surfaces: IDE, CLI, and GitHub.com. The philosophy is "meet developers where they are" with the broadest distribution of any AI coding tool. The CLI (GA Feb 2026) brings terminal-native agentic coding to Copilot subscribers.

### Feature Development Pipeline

- **Agent Mode (IDE):** Autonomous planning, implementation, and iteration within VS Code.
- **Copilot CLI:** Full terminal agent with specialized built-in agents (Explore, Task, Code Review, Plan). Autopilot mode for end-to-end autonomous work.
- **Coding Agent (cloud):** Background delegation with `&` prefix. Creates PRs from issues.
- **Copilot Workspace:** Browser-based agent for planning and implementing from GitHub issues (being merged into core Copilot).

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **.github/copilot-instructions.md** | Repo-level instructions |
| **Custom instructions** | User and org-level prompts |
| **MCP support** | Built-in GitHub MCP server + custom servers |
| **Plugins** | Community and custom plugins from GitHub repos |

### Sub-Agent Capabilities

- **Built-in specialized agents:** Explore (codebase analysis), Task (builds/tests), Code Review (change review), Plan (implementation planning). Copilot auto-delegates to appropriate agents.
- **Background delegation:** Offload to cloud coding agent, resume with `/resume`.
- **Plugin system:** Extend with community agents.

### Strengths

- Broadest distribution (GitHub integration, VS Code default, CLI)
- Model-agnostic with best-in-class model selection
- Built-in specialized agents are well-designed
- GitHub ecosystem integration (issues, PRs, Actions)
- Most affordable pricing

### Weaknesses

- Configuration is less expressive than Claude Code (no hooks, no lifecycle events)
- "Jack of all trades" -- doesn't lead in any single capability
- Agent mode quality depends heavily on model choice
- Enterprise features lag behind Cursor

### Community & Maturity

Largest user base of any AI coding tool (installed in nearly every VS Code instance). Strong enterprise adoption via GitHub. CLI community growing since GA in Feb 2026.

**Sources:**
- [Copilot CLI GA](https://github.blog/changelog/2026-02-25-github-copilot-cli-is-now-generally-available/)
- [Agent Mode 101](https://github.blog/ai-and-ml/github-copilot/agent-mode-101-all-about-github-copilots-powerful-mode/)
- [Copilot CLI Enhanced Agents](https://github.blog/changelog/2026-01-14-github-copilot-cli-enhanced-agents-context-management-and-new-ways-to-install/)
- [Copilot Features](https://docs.github.com/en/copilot/get-started/features)

---

## 8. Aider

**Surface:** Terminal CLI  
**Model:** Model-agnostic (works with Claude, GPT, Gemini, local models)  
**Open source:** Fully open-source (Apache 2.0)  
**GitHub stars:** 30K+

### Core Philosophy

Aider pioneered the **architect/editor** dual-model pattern: a "main model" (architect) reasons about the solution, and an "editor model" translates the plan into precise file edits. This separation produces state-of-the-art benchmark results by letting each model do what it's best at. Aider is the most model-agnostic tool in the market.

### Feature Development Pipeline

- **Chat modes:** `code` (direct editing), `architect` (plan then edit), `ask` (discussion only), `help` (docs)
- **Auto-commits:** Aider commits each change with descriptive messages, creating clean git history
- **Lint and test integration:** Run linters and tests after every change, auto-fix failures

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **CONVENTIONS.md** | Repo-level coding standards and conventions |
| **.aider.conf.yml** | Project-level configuration (model, edit format, git settings) |
| **~/.aider.conf.yml** | User-level defaults |
| **Command-line flags** | Per-session overrides |

### Sub-Agent Capabilities

None. Aider is a single-agent tool by design. It can use the architect/editor pattern (two models cooperating), but this is model orchestration, not agent orchestration.

### Strengths

- Best model-agnostic support (works with any OpenAI-compatible API, local models)
- Architect/editor pattern is elegant and produces strong benchmark results
- Git-native: auto-commits, clean history, easy to review
- Fully open-source with active development
- Lowest barrier to entry for terminal-based agentic coding
- CONVENTIONS.md is simple and effective

### Weaknesses

- No sub-agent or multi-agent capabilities
- No hooks or lifecycle events
- Configuration is simpler than Claude Code (good and bad)
- Single-session only, no background/async execution
- Smaller feature set than Claude Code or Cursor

### Community & Maturity

Strong open-source community. 30K+ GitHub stars. Active blog with benchmark results. Community-contributed conventions repository. Paul Gauthier (creator) is a respected voice in the agentic coding space.

**Sources:**
- [Aider Chat Modes](https://aider.chat/docs/usage/modes.html)
- [Architect/Editor Pattern](https://aider.chat/2024/09/26/architect.html)
- [Aider Documentation](https://aider.chat/docs/)

---

## 9. Cline

**Surface:** VS Code extension (primary), CLI (2.0 with parallel agents)  
**Model:** Model-agnostic  
**Open source:** Fully open-source  
**GitHub stars:** Fastest-growing AI open-source project (Octoverse 2025)

### Core Philosophy

Cline is a local-first, model-agnostic autonomous agent that runs inside VS Code with explicit human-in-the-loop approval for every action. Its philosophy is maximum transparency: you see and approve every file change and terminal command. Plan/Act modes separate reasoning from execution.

### Feature Development Pipeline

- **Plan mode:** Agent reasons about approach without making changes
- **Act mode:** Agent executes with approval gates for each action
- **Browser integration:** Can interact with web apps for testing
- **CLI 2.0 (Feb 2026):** Parallel agents and headless CI/CD integration

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **.clinerules** | Repo-level instructions |
| **MCP integration** | Extensive MCP server support |
| **Custom system prompts** | Per-session behavior customization |

### Sub-Agent Capabilities

- **CLI 2.0:** Parallel agents for CI/CD workflows
- **MCP-based tool extension** rather than sub-agent delegation
- Simpler model than Claude Code's sub-agent architecture

### Strengths

- Best transparency/approval model (see and approve everything)
- Fully open-source and model-agnostic
- Plan/Act separation is intuitive
- Strong MCP ecosystem
- 5M+ developers

### Weaknesses

- Approval fatigue in complex tasks (every action needs approval)
- Less mature configuration than Claude Code
- VS Code only
- No built-in spec or design workflow

### Community & Maturity

5M+ users. Fastest-growing AI open-source project. $32M Series A funding. Very active development.

**Sources:**
- [Cline.bot](https://cline.bot)
- [Cline GitHub](https://github.com/cline/cline)
- [Cline Review 2026](https://vibecoding.app/blog/cline-review-2026)

---

## 10. Devin (Cognition)

**Surface:** Web app, Slack, IDE integrations  
**Model:** Proprietary  
**Pricing:** Core ($20/mo), Teams ($40/mo), Enterprise (custom)

### Core Philosophy

Devin is the most autonomous agent in the market. It doesn't assist with coding -- it *does* the coding. Give it a task (via Slack, web, or API), and it plans, implements, tests, debugs, and delivers working software. The mental model is "a junior engineer you can assign tickets to" rather than "an AI pair programmer."

### Feature Development Pipeline

Devin handles the full lifecycle autonomously:
1. Receives task (natural language, issue, Slack message)
2. Plans implementation
3. Writes code, tests, and documentation
4. Debugs and iterates until tests pass
5. Creates PR for human review
6. Can test desktop apps via computer use

### Configuration Mechanisms

Limited compared to developer-facing tools. Devin learns from feedback on its outputs rather than from configuration files. Repository context is ingested automatically.

### Sub-Agent Capabilities

- Infinitely parallelizable: run dozens of Devin instances on different tasks simultaneously
- Each instance is independent (not sub-agents of a parent)
- No user-configurable sub-agent orchestration

### Strengths

- Most autonomous: genuinely completes tasks end-to-end
- Parallel execution at scale
- Good for well-defined, repeatable tasks (migrations, test coverage, small tickets)
- Enterprise adoption (Goldman Sachs, etc.)
- Computer use for end-to-end testing

### Weaknesses

- Poor at tasks requiring mid-stream requirement changes
- Not suitable for complex architectural work
- Black box: limited visibility into decision-making
- No developer-facing configuration system
- Best for junior-level tasks (4-8 hour scope)

### Community & Maturity

Enterprise-focused. Cognition Labs raised significant funding. The acquisition of Windsurf signals ambition to cover both autonomous and assisted coding. Production use at major enterprises.

**Sources:**
- [Devin Docs](https://docs.devin.ai/release-notes/overview)
- [Devin 2025 Performance Review](https://cognition.ai/blog/devin-annual-performance-review-2025)
- [Devin Wikipedia](https://en.wikipedia.org/wiki/Devin_AI)

---

## 11. Factory

**Surface:** Terminal, IDE, Slack, Linear, browser, custom scripts  
**Model:** LLM-agnostic  
**Pricing:** Enterprise (custom)

### Core Philosophy

Factory builds "Droids" -- specialized autonomous agents for different parts of the development lifecycle. Unlike general-purpose tools, Factory breaks the pipeline into distinct agents: CodeDroid (implementation), Review Droid (PRs), QA Droid (testing), and others. The philosophy is that specialized agents outperform general-purpose ones.

### Feature Development Pipeline

- **CodeDroid:** Feature implementation from specs
- **Review Droid:** Automated code review on PRs
- **QA Droid:** Test generation and execution
- **Additional Droids:** Documentation, incident response, codebase Q&A

### Configuration Mechanisms

Enterprise-configured. Less developer-facing configuration than Claude Code or Cursor.

### Sub-Agent / Multi-Agent Capabilities

Factory's entire architecture is multi-agent by design. Droids are specialized agents that can be composed. #1 on Terminal-Bench (58.75%).

### Strengths

- Specialized agents outperform generalists on their specific tasks
- Interface-agnostic (terminal, IDE, Slack, Linear)
- Strong enterprise customers (NVIDIA, MongoDB, Zapier, Bayer)
- Top Terminal-Bench performance

### Weaknesses

- Enterprise-only pricing
- Less suitable for individual developers
- Limited public documentation on configuration
- Smaller community than open-source tools

### Community & Maturity

Enterprise-focused. $50M Series B. Strong customer list but smaller developer community.

**Sources:**
- [Factory.ai](https://factory.ai/)
- [Factory Terminal-Bench #1](https://factory.ai/news/terminal-bench)
- [Factory Droids Launch](https://siliconangle.com/2025/09/25/factory-unleashes-droids-software-agents-50m-fresh-funding/)

---

## 12. Augment Code

**Surface:** IDE extension, CLI, desktop app (Augment Intent)  
**Model:** Custom + multi-model  
**Pricing:** Free tier, Pro, Enterprise

### Core Philosophy

Augment's differentiator is its **Context Engine** -- a system that maintains a live understanding of your entire codebase including code, dependencies, architecture, and git history. While other tools read files on demand, Augment indexes everything and maintains a persistent semantic model. The bet is that better context produces better code, especially in large codebases.

### Feature Development Pipeline

- **Augment Intent (desktop app):** Spec-driven development with multi-agent orchestration. Multiple agents share a living spec and coordinate around a single evolving plan.
- **IDE extension:** Standard AI-assisted coding with superior context
- **CLI:** Terminal agent with Augment's context engine

### Configuration Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **MCP support** | Published as an MCP provider, enabling other tools (Claude Code, Cursor, Codex) to use Augment's context engine |
| **Project settings** | IDE-level configuration |

### Sub-Agent / Multi-Agent Capabilities

- **Augment Intent:** Multi-agent orchestration around a shared, evolving spec. Agents coordinate rather than work independently.
- **MCP provider:** Other agents can use Augment's context as a tool, making it a force multiplier for Claude Code or Cursor.

### Strengths

- Best-in-class codebase understanding (Context Engine)
- MCP provider model lets it enhance other tools
- Augment Intent's shared-spec multi-agent model is unique
- Works well with large, complex codebases
- Claims 70%+ improvement in agentic coding performance across Claude Code, Cursor, and Codex

### Weaknesses

- Smaller market share than top tools
- Context Engine is proprietary (vendor lock-in)
- Intent desktop app is macOS only
- Less community content and ecosystem

### Community & Maturity

Growing. Backed by significant funding. Positioning as infrastructure ("the context layer") rather than a standalone tool. Adoption accelerating through MCP integration.

**Sources:**
- [Augment Code](https://www.augmentcode.com/)
- [Augment MCP Launch](https://siliconangle.com/2026/02/06/augment-code-makes-semantic-coding-capability-available-ai-agent/)
- [Augment Review (New Stack)](https://thenewstack.io/augment-code-an-ai-coding-tool-for-real-development-work/)

---

## 13. Comparison Matrix

| Tool | Surface | Config System | Hooks/Events | Sub-Agents | Multi-Agent | Spec/Design | Autonomy Level | Open Source | Best For |
|------|---------|--------------|--------------|------------|-------------|-------------|----------------|------------|----------|
| **Claude Code** | Terminal | CLAUDE.md + hooks + skills + subagents | 24 events, 3 handler types | Yes (isolated context) | Agent Teams | Via custom skills | High (with approval) | Partial | Senior engineers who want full control |
| **Codex** | Cloud + CLI | AGENTS.md + config.toml | None | No | Parallel cloud tasks | No | Very high (cloud) | CLI only | Batch autonomous tasks, security-conscious orgs |
| **Cursor** | IDE | .cursorrules + skills | Automations (event-triggered) | No | 8 parallel agents (3.0) | No | High | No | Visual workflow, team collaboration |
| **Windsurf** | IDE | Rules + skills + flows | File events | No | Limited | No | Medium-High | No | Flow-aware assisted coding |
| **Antigravity** | IDE + Manager | Project rules + skills | No | No | Best orchestration UI | Via artifacts | High | No | Multi-agent orchestration, Google ecosystem |
| **Kiro** | IDE | Steering files + hooks | File/workspace events | Autonomous agent | Single persistent | **Best** (native spec pipeline) | Very high (days) | Yes | Spec-driven development, regulated industries |
| **Copilot** | IDE + CLI + GitHub | Instructions.md + MCP | No | Built-in specialized | Background agent | No | Medium-High | No | Broadest reach, GitHub-native teams |
| **Aider** | Terminal | CONVENTIONS.md + yaml | None | No | No | No | Medium | Yes (Apache 2.0) | Model-agnostic terminal work, local models |
| **Cline** | VS Code + CLI | .clinerules + MCP | No | CLI 2.0 parallel | CLI 2.0 | No | Medium (approval gates) | Yes | Transparency, open-source advocates |
| **Devin** | Web/Slack | Minimal | No | No | Parallel instances | No | **Highest** | No | Well-defined tickets, batch work |
| **Factory** | Multi-surface | Enterprise config | No | Specialized Droids | **Native** multi-agent | No | Very high | No | Enterprise automation at scale |
| **Augment** | IDE + CLI + Desktop | MCP provider | No | Intent multi-agent | Shared-spec coordination | Intent (spec-driven) | Medium-High | No | Large codebases, context-heavy work |

---

## 14. Assessment: What Matters for Quality-Focused Teams

### The Configuration Standard War

Three repo-level instruction file formats have emerged: **CLAUDE.md** (Claude Code), **AGENTS.md** (Codex), and **.cursorrules** (Cursor). Of these, CLAUDE.md has the most expressive ecosystem (hooks, skills, subagents, rules directories). The **Agent Skills** open standard is being adopted across Claude Code, Codex, Cursor, and Windsurf, creating the first cross-tool portability for workflows.

**Recommendation for downstream agents:** Design configuration that can target CLAUDE.md as the primary format with adapters for AGENTS.md and .cursorrules. Skills should follow the Agent Skills standard for portability.

### Tool Selection by Engineering Philosophy

**For teams that value control and determinism:**
- **Claude Code** is the clear winner. Its 24 lifecycle hook events with deterministic execution, explicit permission model, and layered configuration give the most precise control over agent behavior. The terminal-native approach composes with existing CI/CD and unix tooling.

**For teams that value visual workflow and speed:**
- **Cursor 3.0** is the leader. Multi-agent orchestration, fast Composer model, and design mode create a productive visual workflow. Self-hosted agents address enterprise concerns.

**For teams that value spec-driven development:**
- **Kiro** is uniquely positioned. No other tool has a native spec-first pipeline. For regulated industries or teams that insist on requirements before code, Kiro is the only tool that enforces this.

**For teams that value autonomous task completion:**
- **Devin** for well-scoped tickets. **Codex cloud** for batch work in sandboxes. **Kiro autonomous agent** for long-running tasks.

**For teams that value multi-agent orchestration:**
- **Antigravity** has the best orchestration UI. **Cursor 3.0** has the best IDE-integrated orchestration. **Claude Code Agent Teams** have the best terminal-based orchestration. **Factory** has the best enterprise multi-agent story.

### The Emerging Stack Pattern

The most sophisticated teams in 2026 use multiple tools:

1. **Claude Code** (or Copilot CLI) for complex, context-heavy work requiring precise control
2. **Cursor** (or Antigravity/Windsurf) as the daily driver IDE for visual editing
3. **Devin** (or Codex cloud) for parallelizable autonomous tasks
4. **Augment** as a context layer enhancing all of the above via MCP

### Key Trends Shaping 2026

1. **Agent Skills standard** is creating cross-tool portability (Claude Code, Codex, Cursor, Windsurf all support it)
2. **MCP (Model Context Protocol)** is becoming the universal integration layer
3. **Background/async agents** are standard (Cursor, Copilot, Codex, Devin, Kiro all support them)
4. **Spec-driven development** is gaining traction as teams learn that "vibe coding" produces technical debt
5. **Self-hosted agents** are an enterprise requirement (Cursor, Codex, Factory offer this)
6. **Context engineering** (how well the tool understands your codebase) is the key differentiator, with Augment leading on pure context quality

### What's Missing Everywhere

No tool has fully solved:
- **Architectural consistency across sessions** (each new session starts fresh)
- **Learning from past mistakes** (agents repeat the same errors)
- **Cross-repo understanding** (monorepo support is improving but still weak)
- **Cost predictability** (token costs vary wildly by task complexity)
- **Reliable spec-to-implementation fidelity** (specs drift during implementation)

---

*Last updated: April 3, 2026*
