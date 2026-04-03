# Spec-Driven Development in Agentic Coding

Research compiled April 2026. Covers how top teams use specifications to steer AI coding agents, the major tools and frameworks, configuration-file-as-spec patterns, and critical analysis.

---

## 1. The Spec-Driven Development Movement

### Definition

Spec-driven development (SDD) is "a development paradigm that uses well-crafted software requirement specifications as prompts, aided by AI coding agents, to generate executable code" ([Thoughtworks](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices)). It emerged in 2025 as the antidote to "vibe coding" -- unstructured prompting that produces code nobody fully understands.

### Three Maturity Levels

Birgitta Bockeler (Thoughtworks/Martin Fowler) identifies three levels ([Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)):

1. **Spec-first**: A spec is written before implementation, then used to guide the agent. The spec may be abandoned after the feature ships.
2. **Spec-anchored**: The spec persists through feature evolution and maintenance. Code and spec stay synchronized.
3. **Spec-as-source**: The spec IS the primary artifact. Humans edit only the spec; code is entirely AI-generated and marked "DO NOT EDIT."

Most teams today practice spec-first. Spec-as-source is aspirational and explored primarily by Tessl.

### Core Workflow Pattern

Across all tools, the pattern converges on four phases:

1. **Specify** -- Capture requirements in natural language, refined into structured specs
2. **Plan** -- Produce a technical design/architecture document
3. **Task** -- Decompose into small, reviewable, testable work items
4. **Implement** -- Agent executes tasks sequentially; human reviews at checkpoints

This pattern appears independently in Kiro, GitHub Spec Kit, JetBrains Junie, BMAD Method, and practitioner workflows with Claude Code.

---

## 2. Major Tools and Frameworks

### 2.1 Kiro (Amazon/AWS)

**What it is**: A VS Code fork powered by Anthropic's Claude, released July 2025. Available at $20/month. ([InfoQ](https://www.infoq.com/news/2025/08/aws-kiro-spec-driven-agent/), [kiro.dev](https://kiro.dev/))

**Three-phase workflow**:
- **Requirements**: Developer describes intent in natural language. Kiro generates user stories with acceptance criteria specifying behaviors, validation rules, and error handling.
- **Design**: Produces architectural documentation with diagrams and data schemas.
- **Tasks**: Decomposes requirements into sequential, trackable coding tasks.

**Bidirectional sync**: Kiro keeps specs synchronized with evolving code. Developers can author code and ask Kiro to update specs, or update specs to refresh tasks.

**Steering files** ([Kiro Docs](https://kiro.dev/docs/steering/)):
- Stored in `.kiro/steering/` (workspace) or `~/.kiro/steering/` (global)
- Three foundational files: `product.md` (purpose, users, features), `tech.md` (frameworks, constraints), `structure.md` (file organization, naming)
- Inclusion modes: Always (default), Conditional (fileMatch glob), Manual (`#steering-file-name`), Auto (matched by purpose)
- Also recognizes AGENTS.md files

**Hooks** ([Kiro Docs](https://kiro.dev/docs/hooks/)):
- Event-driven automations triggered by file saves, agent turns, tool invocations, spec execution, or manual activation
- Can execute agent prompts or shell commands
- Configured via natural language or structured form
- Example: validate UI changes against Figma specs on file save

**Critical assessment**: Martin Fowler's team found Kiro struggles with problem sizing -- small bugs become 4 user stories with 16 acceptance criteria. Best suited for greenfield projects with clear scope. ([Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html))

### 2.2 Google Antigravity

**What it is**: Google's agentic IDE, announced November 2025 alongside Gemini 3. Free in public preview. A VS Code fork with agent-first architecture. ([Google Developers Blog](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/))

**Key differentiator**: Adaptive spec generation rather than rigid workflows. The model assesses task complexity and autonomously decides what level of planning rigor is needed. Simple tasks skip elaborate planning; complex tasks trigger detailed implementation plans.

**Artifacts system**:
- Task lists, implementation plans, walkthroughs, screenshots, browser recordings
- Artifacts are collaborative -- humans leave feedback directly on them
- Knowledge Base provides persistent memory of project patterns

**Configuration**:
- `agents.md` for specialized AI personas
- `skills/` directory with `.md` files defining strict technical rules
- Custom slash-command workflows
- Supports Gemini 3 Pro, Claude Sonnet 4.6, and OpenAI models

**Manager view**: A control center for orchestrating multiple agents working in parallel across workspaces. Developer transitions from "writer of code" to "architect" or "mission controller."

([Google Cloud Medium](https://medium.com/google-cloud/benefits-and-challenges-of-spec-driven-development-and-how-antigravity-is-changing-the-game-3343a6942330), [Bay Tech Consulting](https://www.baytechconsulting.com/blog/google-antigravity-ai-ide-2026))

### 2.3 GitHub Spec Kit

**What it is**: Open-source CLI toolkit for spec-driven development. Works with GitHub Copilot, Claude Code, and Gemini CLI. ([GitHub Blog](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/), [GitHub repo](https://github.com/github/spec-kit))

**Four-phase workflow**:
1. `/specify` -- Generate detailed spec from high-level description
2. `/plan` -- Create technical implementation plan with stack/architecture constraints
3. `/tasks` -- Break into small, reviewable, testable work items
4. `/implement` -- Agent executes tasks

**Constitutional rules**: Foundation files enforced across all changes, establishing organizational standards and compliance requirements.

**Critical assessment**: Creates numerous markdown files with checklists. Creates branches per change request. Review overhead of verbose markdown documentation can be counterproductive -- Fowler notes preferring code review to specification review. ([Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html))

### 2.4 Tessl

**What it is**: An AI-native development platform aspiring to spec-as-source level SDD. Spec Registry is free; Framework is in beta. ([tessl.io](https://tessl.io/))

**Distinctive approach**:
- Generated code marked with `// GENERATED FROM SPEC - DO NOT EDIT`
- Currently 1:1 spec-to-file mapping
- Uses `@generate` and `@test` tags in specs
- Single source of truth for skills and context, reusable across agents and environments

**Challenge**: Non-determinism remains an issue even with identical specs. Requires iterative spec refinement. ([Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html))

### 2.5 JetBrains Junie

**What it is**: JetBrains' AI coding agent with a structured spec-driven approach. ([JetBrains Blog](https://blog.jetbrains.com/junie/2025/10/how-to-use-a-spec-driven-approach-for-coding-with-ai/))

**Workflow**:
1. `requirements.md` -- High-level goals and outcomes
2. `plan.md` -- Implementation strategy (use "Think More" for deeper analysis)
3. `tasks.md` -- Enumerated checklist with completion checkboxes
4. Controlled execution in bounded phases

**Guidelines**: `.junie/guidelines.md` stores technical instructions.

**Key principle**: "Don't ask Junie to do everything in tasks.md in one go. Instead, start with a subset."

### 2.6 BMAD Method

**What it is**: "Breakthrough Method for Agile AI-Driven Development." Open-source framework combining SDD with human-in-the-loop governance. Works with VS Code, Cursor, Claude Code. ([GitHub](https://github.com/bmad-code-org/BMAD-METHOD), [docs](https://docs.bmad-method.org/))

**Multi-agent personas**: Product Manager, Architect, Developer, Scrum Master, UX Designer -- each defined as "Agent-as-Code" markdown files with expertise, responsibilities, constraints, and expected outputs.

**Four-phase cycle**: Analysis (one-page PRD) -> Planning (user stories with acceptance criteria) -> Solutioning (architecture) -> Implementation.

**Philosophy**: Documentation (PRDs, architecture designs, user stories) is the source of truth, not just source code.

---

## 3. Rules Files as Implicit Specifications

Every major coding agent now supports project-level configuration files that function as persistent specifications. These have converged toward a common pattern: markdown files providing conventions, constraints, and context.

### 3.1 The Ecosystem

| Tool | File | Location | Scope |
|------|------|----------|-------|
| Claude Code | `CLAUDE.md` | Project root + subdirs | Hierarchical, auto-merged |
| GitHub Copilot | `AGENTS.md` | Root + nested dirs | Nearest-file-wins |
| GitHub Copilot | `.github/copilot-instructions.md` | `.github/` | Project-wide |
| Cursor | `.cursor/rules/*.md` | `.cursor/rules/` | Per-rule scoping |
| Gemini CLI | `GEMINI.md` | Root + `~/.gemini/` | Hierarchical |
| Kiro | `.kiro/steering/*.md` | `.kiro/steering/` + `~/` | Always/Conditional/Manual/Auto |
| Windsurf | Rules/Rulebooks | IDE config | Slash-command invocation |
| Aider | `CONVENTIONS.md` | Loaded via `--read` | Session-scoped |
| JetBrains Junie | `.junie/guidelines.md` | `.junie/` | Project-wide |
| Antigravity | `agents.md` + `skills/*.md` | Root + `skills/` | Per-agent + per-skill |

### 3.2 AGENTS.md Standard

AGENTS.md is emerging as a cross-tool standard, stewarded by the Agentic AI Foundation under the Linux Foundation. It originated from collaborative efforts across OpenAI Codex, Amp, Jules (Google), Cursor, and Factory. ([agents.md](https://agents.md/), [GitHub Blog](https://github.blog/changelog/2025-08-28-copilot-coding-agent-now-supports-agents-md-custom-instructions/))

Supported by: GitHub Copilot, Kiro, Gemini CLI (via config), and most agent frameworks.

### 3.3 CLAUDE.md Patterns

From analysis of production CLAUDE.md files ([Claude Code Docs](https://code.claude.com/docs/en/best-practices), [eesel.ai](https://www.eesel.ai/blog/claude-code-best-practices)):

**What works**:
- Common bash commands (build, test, lint, deploy)
- Code style guidelines with concrete examples
- Key architectural patterns and file locations
- Testing instructions and framework setup
- Hierarchical files: root CLAUDE.md + subdirectory overrides

**Advanced patterns**:
- Subagents in `.claude/agents/` for delegated specialized tasks
- Skills as lazy-loaded context
- Context management: compact at 70%, clear at 90%

### 3.4 Best Practices Across All Tools

GitHub's analysis of 2,500+ repositories identified six essential areas for agent configuration files ([GitHub Blog](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)):

1. **Commands**: Executable commands with flags (`npm test`, `pytest -v`)
2. **Testing**: Framework, file locations, coverage expectations
3. **Project structure**: Exact directory layout
4. **Code style**: Real code examples (one snippet beats three paragraphs)
5. **Git workflow**: Branch naming, commit format, PR requirements
6. **Boundaries**: Three-tier system:
   - Always: safe actions, no approval needed
   - Ask first: high-impact changes requiring review
   - Never: absolute prohibitions (e.g., never commit secrets)

**Anti-patterns**: "Most agent files fail because they're too vague" -- say "React 18 with TypeScript" not "React project." The "curse of instructions" shows excessive directives reduce adherence to each.

---

## 4. How to Write a Good Spec for AI Agents

Addy Osmani's guide ([AddyOsmani.com](https://addyosmani.com/blog/good-spec/)) synthesizes best practices:

### Structure

```markdown
# Project Spec: [Name]

## Objective
[Clear goal statement]

## Tech Stack
[Specific versions and dependencies]

## Commands
[Build, test, lint with exact flags]

## Project Structure
[Directory layout]

## Boundaries
[Always/Ask/Never framework]
```

### Five Principles

1. **Start high-level, let AI expand**: Begin with a concise product brief. Have the agent generate detailed specs. Save as persistent `SPEC.md`.

2. **Structure like a professional document**: Cover the six essential areas. Use the three-tier boundary system.

3. **Break into modular prompts**: Avoid monolithic specs. Feed only relevant sections per task. Consider subagents for different domains.

4. **Build in self-checks**: Include verification steps. Embed conformance testing with expected input/output. Inject domain expertise and known pitfalls.

5. **Test, iterate, evolve**: Run tests after each milestone. Update specs when requirements change. Version-control specs alongside code.

### What Makes Specs Effective (Thoughtworks)

- Use domain-oriented ubiquitous language (business intent, not technology)
- Follow clear structures (Given/When/Then scenarios)
- Achieve completeness yet conciseness
- Maintain clarity and determinism to minimize hallucinations
- Incorporate semi-structured, machine-readable formats alongside natural language

---

## 5. Context Engineering: The Broader Frame

Spec-driven development is a subset of a larger discipline emerging in 2025-2026: **context engineering** -- "curating what the model sees so that you get a better result." ([Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html))

### Context Categories for Coding Agents

**Reusable prompts** (two types):
- *Instructions*: Task-oriented prompts directing specific actions
- *Guidance*: General conventions and guardrails

**Context interfaces** (how agents access information):
- Built-in tools (bash, file search)
- MCP Servers (custom data access programs)
- Skills (on-demand resources, lazy-loaded)

### Configuration Feature Taxonomy (Claude Code example)

| Feature | Type | Loading | Use Case |
|---------|------|---------|----------|
| CLAUDE.md | Guidance | Always | Project conventions |
| Rules | Guidance | Path-based | Scoped, modular guidance |
| Skills | Guidance/Instructions | On-demand | Task-specific resources |
| Subagents | Instructions + Config | On-demand | Specialized parallel tasks |
| MCP Servers | Data access | Tool calls | API/external integration |
| Hooks | Scripts | Lifecycle events | Deterministic automation |

### Critical Insight

"Larger context windows don't justify indiscriminate information dumping. Effectiveness decreases with excessive context." Start minimal, expand based on actual needs. The "curse of instructions" applies: more rules can mean less adherence to each.

---

## 6. Critical Analysis

### What Works

- **Forcing upfront thinking**: The primary value of SDD is that it forces requirements analysis and design before coding. This is genuinely valuable regardless of tooling.
- **Persistent context**: Rules files (CLAUDE.md, AGENTS.md, etc.) eliminate repetitive explanation and create team consistency.
- **Reviewable intent**: Specs make the "what" and "why" visible and reviewable before implementation begins.
- **Bounded execution**: Task lists with checkpoints prevent agents from going off the rails on large changes.

### What Doesn't Work

- **Problem-size mismatch**: All current SDD tools impose heavyweight workflows unsuitable for small bugs or minor features. Kiro turns a small fix into 4 user stories with 16 acceptance criteria. ([Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html))
- **Review burden**: Creating extensive markdown for review may be worse than reviewing code directly. Fowler: "I prefer code review to verbose specification documentation."
- **Control illusion**: Despite elaborate templates and checklists, agents frequently ignore instructions or misinterpret context, creating duplicates or over-interpreting requirements.
- **Non-determinism**: Even with identical specs, generated code varies between runs. This complicates maintenance and upgrades.
- **Waterfall echoes**: Critics note SDD "revives the old idea of heavy documentation before coding -- an echo of the Waterfall era" though proponents argue iteration loops remain shorter. ([Marmelab](https://marmelab.com/blog/2025/11/12/spec-driven-development-waterfall-strikes-back.html))
- **Semantic diffusion**: "Spec" already conflates detailed prompts with comprehensive specifications, creating definitional confusion.

### The Antigravity Counter-Model

Google's Antigravity represents a different philosophy: adaptive rigor. Instead of always running through specify->plan->task->implement, the model decides how much planning each task requires. Simple tasks skip specs; complex ones get full treatment. This may be more aligned with how experienced developers actually work.

### DORA Research

The 2025 DORA report ("State of AI-assisted Software Development") found that "AI's primary role is as an amplifier, magnifying an organization's existing strengths and weaknesses." Teams with good engineering practices benefit more from AI tools; teams with poor practices see those problems amplified. ([DORA](https://dora.dev/research/2025/dora-report/))

---

## 7. Practical Implications for Skill Design

### What a quality-focused toolkit should take from SDD

1. **Spec generation should be proportional to task complexity**. A bug fix needs a one-liner description; a new feature needs a proper spec. Avoid one-size-fits-all ceremony.

2. **Specs should decompose by capability, not by layer**. Each unit of work delivers something a user or system can do that it couldn't before. (Matches the commit philosophy in the user's CLAUDE.md.)

3. **Rules files are the foundation**. CLAUDE.md/AGENTS.md patterns are the highest-leverage investment. They provide persistent, low-overhead context engineering.

4. **The six essential areas** (commands, testing, structure, style, git workflow, boundaries) should be the minimum checklist for any project's rules file.

5. **Three-tier boundaries** (Always/Ask/Never) are a proven pattern across all tools. Build this into skill design.

6. **Hooks for deterministic quality gates**. Use lifecycle hooks (file save, pre-commit, agent turn complete) to enforce non-negotiable standards without relying on the LLM to remember them.

7. **Bidirectional spec-code sync is aspirational but premature**. Kiro's approach is interesting but the tooling isn't reliable enough yet. For now, treat specs as planning artifacts that inform but don't constrain.

8. **Context engineering > spec engineering**. The broader discipline of curating what the model sees (and when) matters more than any specific spec format. Size management, scoped loading, and lazy context are the real levers.

---

## Sources

### Kiro
- [Kiro Homepage](https://kiro.dev/)
- [Kiro Steering Docs](https://kiro.dev/docs/steering/)
- [Kiro Hooks Docs](https://kiro.dev/docs/hooks/)
- [InfoQ: Beyond Vibe Coding](https://www.infoq.com/news/2025/08/aws-kiro-spec-driven-agent/)
- [RedMonk: Al Harris on Kiro](https://redmonk.com/videos/al-harris-on-kiro-and-spec-driven-development/)

### Google Antigravity
- [Google Developers Blog](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/)
- [Antigravity and SDD (Medium)](https://medium.com/google-cloud/benefits-and-challenges-of-spec-driven-development-and-how-antigravity-is-changing-the-game-3343a6942330)
- [Bay Tech Consulting Review](https://www.baytechconsulting.com/blog/google-antigravity-ai-ide-2026)

### GitHub Spec Kit
- [GitHub Blog: Spec-Driven Development](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [GitHub Spec Kit Repo](https://github.com/github/spec-kit)

### AGENTS.md & Rules Files
- [AGENTS.md Standard](https://agents.md/)
- [GitHub: Writing Great agents.md](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
- [GitHub Changelog: AGENTS.md Support](https://github.blog/changelog/2025-08-28-copilot-coding-agent-now-supports-agents-md-custom-instructions/)
- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Cursor Rules Docs](https://cursor.com/docs/context/rules)
- [GEMINI.md Docs](https://geminicli.com/docs/cli/gemini-md/)

### Spec-Driven Development Analysis
- [Martin Fowler: SDD Tools (Kiro, Spec-Kit, Tessl)](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)
- [Martin Fowler: Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)
- [Thoughtworks: Spec-Driven Development](https://www.thoughtworks.com/en-us/insights/blog/agile-engineering-practices/spec-driven-development-unpacking-2025-new-engineering-practices)
- [Addy Osmani: How to Write a Good Spec](https://addyosmani.com/blog/good-spec/)

### Other Frameworks
- [Tessl](https://tessl.io/)
- [JetBrains Junie Spec Approach](https://blog.jetbrains.com/junie/2025/10/how-to-use-a-spec-driven-approach-for-coding-with-ai/)
- [BMAD Method](https://github.com/bmad-code-org/BMAD-METHOD)
- [Heeki Park: SDD with Claude Code](https://heeki.medium.com/using-spec-driven-development-with-claude-code-4a1ebe5d9f29)

### Research
- [DORA 2025: State of AI-assisted Software Development](https://dora.dev/research/2025/dora-report/)
- [Marmelab: SDD - Waterfall Strikes Back](https://marmelab.com/blog/2025/11/12/spec-driven-development-waterfall-strikes-back.html)
- [Claude Blog: Eight Trends in 2026](https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026)
