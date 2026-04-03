# Agentic Development Toolkit: Synthesis Spec (v2)

**Purpose:** Unified specification for building agentic development skills. Written for an agent that will author skills.

**Research base:** 150+ sources across 25 supporting documents (01-25). Includes Anthropic official docs, Martin Fowler/Thoughtworks, Kent Beck, Simon Willison, Addy Osmani, Adam Tornhill/CodeScene, METR, Stripe Engineering, GitHub, Google, the Claude Code source leak analysis, and dozens of practitioner blog posts.

**Evidence quality:** This document distinguishes between strong evidence (multiple independent sources, empirical data), moderate evidence (practitioner consensus without controlled studies), and contested claims (reasonable people disagree). Specific data points are annotated with their source vintage — findings from older models (Claude 3.5/3.7) may not hold for current models (Opus 4.6).

---

## Part 1: Foundational Principles

### P1: Engineering Discipline Increases, Not Decreases
**Evidence: STRONG**

Every serious practitioner agrees: agentic coding requires *more* discipline, not less. AI amplifies existing practices — good and bad (DORA 2025).

- Teams without guardrails see 35-40% increase in bug density within 6 months (Qodo 2025 survey)
- 81% of developers using AI *with* rigorous review report quality improvements vs 55% without (Qodo 2025 survey — note: this is self-reported, not measured)
- The gap is entirely about process

### P2: Spec Before Code — Or Build Then Spec
**Evidence: STRONG (with important exceptions)**

Most practitioners converge on specifying before coding. But the red team identified a legitimate counter-pattern.

**The default (known problems):** Spec first, proportional to complexity:
- **Trivial** (typo fix, config change): No spec. Just do it.
- **Small** (1-2 files, clear scope): One-sentence description + acceptance criteria
- **Medium** (3-5 files): Structured spec with user stories, technical approach, task list
- **Large** (6+ files, new capability): Full spec → design → task decomposition → phased implementation

**The inversion (unknown problems):** When requirements are uncertain, the pipeline inverts:
1. **Spike** — build a throwaway prototype to learn
2. **Spec** — document what you learned (Marmelab's `vibe-spec` tool generates specs from agent logs, post-hoc)
3. **Build properly** — implement for real with tests

**When to invert:** New domain, unfamiliar technology, requirements that will emerge from user feedback, genuinely novel problems. The key signal: if writing a spec feels like guessing, build first.

**Failure modes both ways:** Over-specifying small tasks (Kiro turns a bug fix into 4 user stories). Under-specifying large features (vibe coding produces 47 subtle production bugs over 6 months — AlterSquare).

### P3: Tests as Success Signals (Not Necessarily TDD Ceremony)
**Evidence: STRONG for tests-as-signals, CONTESTED for strict TDD process**

Tests are the single most effective quality mechanism for agentic coding. But the red team found empirical evidence that the *strict red-green-refactor ceremony* can hurt when applied naively to agents.

**What the evidence supports:**
- Tests give agents a binary success signal — the clearest goal possible
- Tests derived from requirements (not implementation) break circular validation
- Targeted test context reduces regressions by 70% (TDAD paper, arXiv 2603.17973)

**What the evidence challenges:**
- TDD *process instructions* ("write tests first, then implement") without specifying *which tests* increased regressions by 64% (TDAD paper)
- Spikes, UI exploration, and novel domains are harmed by test-first (Kent Beck's own "Spike and Stabilize")
- TDD adds 15-35% development time upfront (IBM/Microsoft data) — zero ROI for throwaway prototypes

**The distinction:** Tests as *verification signals* = universally valuable. Strict red-green-refactor *ceremony* = valuable for well-understood problems with clear acceptance criteria, counterproductive for exploration.

**Default behavior for skills:** Test-first for implementation of known requirements. Test-after (in fresh context) for spikes and exploration. Never skip tests entirely.

Note: The Codemanship claim that "DORA data shows elite-performing teams are 'pretty much all doing TDD'" is Codemanship's interpretation, not a direct DORA finding. DORA supports test automation as a capability of elite teams.

### P4: Deterministic Gates for Fast Checks, Advisory Instructions for Semantic Guidance
**Evidence: STRONG (with speed caveat)**

Hooks always fire. CLAUDE.md instructions sometimes get ignored under context pressure.

**But hooks have a speed constraint:** Pre-commit hooks taking >1 second create friction developers route around. A Rails app with a 3-minute test suite cannot run the full suite on every commit.

**The tiered approach:**
- **Pre-commit hooks (must be fast, <1s):** Formatting, targeted lint rules
- **Pre-push hooks (can be slower):** Type checking, focused test suites
- **CI gates (can be slow):** Full test suite, security scans, comprehensive lint
- **Advisory instructions (CLAUDE.md):** Architectural patterns, domain conventions, design philosophy — things no tool can enforce deterministically

**CLAUDE.md budget:** Practitioners report compliance degradation around 150-200 lines, but this is observational, not from controlled measurement. Organization and relevance matter more than raw line count. Use `@path/to/import` syntax for modular instruction sets.

### P5: Selective Context Transfer for Review
**Evidence: MODERATE (refined from original "fresh context" recommendation)**

A model reviewing its own output in the same session carries forward assumptions. But the red team showed total context isolation loses intent — architectural decisions appear arbitrary without the constraints that motivated them.

**Recommended approach — selective context transfer:**
- Pass to the reviewer: the spec, review checklist, high-level design rationale
- Strip from reviewer: implementation conversation history, false starts, debugging tangents
- This preserves intent while removing anchoring

**Review effectiveness hierarchy:**
1. Cross-model review (different vendor) — catches model-specific blind spots
2. Selective-context same-model review — practical default
3. Full fresh-context review — use when you can afford the cost of reconstructing intent
4. Same-session self-review — last resort, catches only mechanical issues

### P6: Capability-Ordered Decomposition (Default, Not Absolute)
**Evidence: STRONG (with acknowledged exceptions)**

Decompose features by capability, not by architectural layer. Each commit delivers one thing a user or system can do that they couldn't before.

**Exception — schema-first decomposition:** For data-heavy work where the schema is the foundation (database-first development, data pipeline work), layer-by-layer decomposition starting from the data model is legitimate. The key test: would this partial commit be meaningful to review and test in isolation?

### P7: Context Is the Fundamental Resource
**Evidence: STRONG**

"Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills." — Anthropic official docs

- Quality degrades at ~60% utilization, not at the limit
- The "Lost in the Middle" problem is real — models focus on beginning and end

**Context management tools:**
- `/clear` between unrelated tasks
- `/btw` for side questions without context growth (ephemeral, no tool access, no history impact)
- `/rewind` and checkpointing for selective conversation/code rollback
- Delegate research to sub-agents (separate context windows)
- After 2 failed corrections, `/clear` and restart with better prompt
- Compact every 25-30 minutes during intensive work

**From the source leak:** Skills sit in the dynamic prompt section. Keeping skill text stable across turns gives up to 5x cost reduction via within-session caching. Avoid injecting timestamps or volatile data into skill text. The practical ceiling for always-loaded skill text is ~2,500 tokens (base system prompt consumes 15-25K).

### P8: Evidence Before Assertions
**Evidence: STRONG**

Before any commit: run typecheck, lint, test suite. Before claiming completion: show actual command output, not assertions. "It should work" is never acceptable.

### P9: The Human Remains the Architect
**Evidence: STRONG (with scalability concern)**

Only senior+ engineers successfully use parallel agents (Anthropic 2026 data). The operating model is "delegate, review, and own."

**The scalability paradox (from red team):** If only seniors succeed with agents + agents replace junior task work + juniors can't develop into seniors through that work = unsustainable pipeline. Employment among software developers aged 22-25 fell nearly 20% between 2022 and 2025. Skills should be designed to be *teachable* — explicit enough that a mid-level engineer can follow the process, not just an expert.

### P10: Git Is the Recovery Mechanism
**Evidence: STRONG**

Frequent atomic commits are save points. Git worktrees provide isolation for parallel work.

**Sub-agent execution models (from source leak):**
- **Fork:** Byte-identical context copy. Cache-efficient — spawning 5 agents costs barely more than 1. Best for parallel read-heavy tasks (reviews, analysis).
- **Teammate:** Separate terminal pane, file-based mailbox. Each gets ~40% context utilization. Best for independent workstreams needing loose coordination.
- **Worktree:** Isolated git branch per agent, auto-cleanup. Best for write-heavy or risky work that might be discarded.

---

## Part 2: The Pipeline

### The Standard Pipeline (Known Problems)

```
SPECIFY → PLAN → IMPLEMENT → REVIEW
```

Ceremony scales with complexity. Trivial tasks skip to implementation. Each phase has inputs, outputs, and gates.

### The Exploration Pipeline (Unknown Problems)

```
SPIKE → LEARN → SPECIFY → IMPLEMENT → REVIEW
```

When you don't know what you're building, build something wrong first to learn. Then spec what you actually need. Marmelab's `vibe-spec` tool formalizes this — generating specs from agent logs, post-hoc.

### The Ralph Loop (Autonomous Iteration)

```
while (incomplete) { read PRD → find next task → implement → update progress → fresh context }
```

The Ralph Loop (named after Ralph Wiggum) runs an agent repeatedly until all PRD items complete. Each iteration starts with fresh context. After each iteration, learnings are written back to AGENTS.md/config files — a working implementation of self-improving agent workflows.

Key properties: fresh context per iteration (prevents context pollution), progress tracking via checklist file, built-in safeguards against infinite loops and cost overruns, can use browser automation for visual verification. Implementations exist from snarktank (original), frankbria (Claude Code), and Vercel Labs (AI SDK).

### Pipeline Selection Guide

| Signal | Pipeline |
|---|---|
| Requirements are clear, code is long-lived | Standard |
| You're guessing at requirements | Exploration (spike first) |
| Many independent, well-scoped tasks | Ralph Loop or parallel sub-agents |
| Throwaway prototype, learning exercise | Skip spec, build fast, discard or spec-after |

---

## Part 3: The Agent Skills Standard

Skills must follow the Agent Skills open standard (agentskills.io). This is not optional — 16+ tools support it (Claude Code, Codex, Cursor, Gemini CLI, Copilot, VS Code, Kiro, Junie, Amp, Goose, and more). See doc 08 for the full spec.

### SKILL.md Format

```yaml
---
name: skill-name          # Required. 1-64 chars. Lowercase + hyphens.
description: >-           # Required. 1-1024 chars. What it does AND when to use it.
  Description here.
---

[Markdown instructions — keep under 500 lines]
```

### Directory Structure

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation (loaded on demand)
└── assets/           # Optional: templates, resources
```

### Progressive Disclosure (Key Architectural Insight)

1. **Metadata (~100 tokens):** `name` and `description` loaded at startup for ALL skills
2. **Instructions (<5,000 tokens):** Full SKILL.md body loaded when skill is activated
3. **Resources (as needed):** Files in scripts/, references/, assets/ loaded only when required

This is why skills can bundle unlimited reference material without bloating every session. Structure heavy docs as separate files in `references/`, not inline in SKILL.md.

### Claude Code Extensions

Claude Code adds frontmatter fields beyond the base spec:
- `disable-model-invocation: true` — skill can only be invoked explicitly (for skills with side effects)
- `user-invocable: true` — appears as a slash command
- `paths: ["**/*.rb"]` — glob-based automatic activation
- `context: fork` — skill runs in a forked context (isolated)

### Custom Subagents (`.claude/agents/`)

Subagents are specialized agents defined as Markdown files with frontmatter. 15+ fields available including `model`, `allowed-tools`, `permission-mode`, `memory`. Subagents cannot spawn their own subagents.

### Plugins

Skills can be distributed as plugins via the `npx skills add <package>` CLI (Vercel Labs). skills.sh is the registry. Major publishers: Microsoft, Vercel, Anthropic.

---

## Part 4: Recommended Skill Set

### Skill 1: Discovery / Brainstorming
**When:** Starting any non-trivial new work
**What:** Structured interview to surface requirements, constraints, edge cases
**Output:** Validated requirements ready for specification
**Key pattern:** "Ask me one question at a time about the hard parts" (Harper Reed)
**Rigidity:** Flexible

### Skill 2: Specification
**When:** After discovery (or after a spike, in the exploration pipeline)
**What:** Generate structured spec proportional to complexity
**Output:** Spec document with user stories, acceptance criteria, boundaries
**Key pattern:** [NEEDS CLARIFICATION] tags for ambiguities; spec length proportional to task size
**Rigidity:** Flexible (ceremony scales with complexity)

### Skill 3: Design / Architecture
**When:** After spec approval, before implementation (medium+ tasks)
**What:** Technical design + capability-ordered task decomposition
**Output:** Design document + ordered task list with dependencies
**Key pattern:** Each task = one capability = one commit. One-sentence test.
**Rigidity:** Rigid on decomposition rules, flexible on design approach

### Skill 4: Implementation
**When:** After plan approval (or directly after spec for small tasks)
**What:** Test-driven implementation with deterministic quality gates
**Output:** Working code + tests, committed atomically
**Key patterns:** Tests as success signals. Test writer isolated from implementation context. Max 3-5 iteration attempts before stopping and reporting. Fresh context per task.
**Mode selection:** Test-first for known requirements. Test-after (fresh context) for spikes/exploration.
**Rigidity:** Rigid on verification gates, flexible on TDD ceremony

### Skill 5: Code Review
**When:** After implementation, before merge
**What:** Selective-context review with falsification
**Output:** Review findings with confidence scores
**Key pattern:** Pass spec + review checklist to reviewer, strip conversation history. Cross-model review when possible.
**Rigidity:** Rigid on context separation, flexible on review dimensions

### Skill 6: Verification / Completion
**When:** Before claiming any work is done
**What:** Run verification commands, confirm output, present evidence. Cross-reference output against original requirements (the gap that caught us: 7 agents missed an explicit user request).
**Output:** Verified passing state with command output as proof
**Rigidity:** Rigid

### Skill 7: Context Management
**When:** Throughout all work
**What:** When to clear, compact, delegate, use /btw, use /rewind
**Output:** N/A (operational guidance)
**Rigidity:** Flexible

### Skill 8: Parallel Execution
**When:** Plan contains 3+ independent tasks
**What:** Orchestrate sub-agents or worktrees for parallel implementation
**Model selection:** Fork for read-heavy (reviews, analysis). Worktree for write-heavy (implementation). Teammate for multi-session orchestration.
**Preconditions (ALL must be met):** 3+ tasks, no shared state, clear file boundaries
**Rigidity:** Rigid on preconditions, flexible on execution

### Skill 9: Self-Improvement / Retrospective
**When:** After completing work (especially after failures or corrections)
**What:** Capture learnings, update rules/skills, verify improvement
**Output:** Updated CLAUDE.md or skill files with new rules
**Key patterns:** Reflection prompt ("Reflect on this mistake. Abstract and generalize. Write to CLAUDE.md."). Human review gate on rule changes. Claude-reflect for semi-automated capture.
**Maturity level:** Most teams are at manual or human-triggered (levels 1-2). Semi-automated tools exist (claude-reflect, pro-workflow). Fully automated is research-only.
**Rigidity:** Flexible

---

## Part 5: Language and Framework Considerations

**Evidence vintage warning:** Most benchmark data is from Claude 3.5/3.7 Sonnet, not current Opus 4.6. Performance gaps likely narrower with current models, but directional findings hold.

### Key Findings

- **AI performance varies dramatically by language.** Multi-SWE-bench (Claude 3.7 Sonnet): Python 52%, Java 22%, TypeScript 2%. Amazon SWE-PolyBench shows smaller gaps (Python 55%, TypeScript 49%). The truth is somewhere between.
- **Type systems are the strongest automated defense.** 94% of LLM compilation errors are type-related (ETH Zurich/UC Berkeley). TypeScript catches these at compile time; Ruby/Python surface them at runtime.
- **Opinionated frameworks beat flexible ones.** Convention-over-configuration aligns with how agents work best. Rails, Next.js, NestJS produce more consistent output than Express, Flask, Sinatra.
- **Codebase health is the single strongest predictor.** AI assistants increase defect risk by 30%+ in code below 7.0 health score. The threshold for reliable AI output is 9.5/10 (CodeScene, peer-reviewed). Industry average is 5.15. Refactor before automating.
- **Java: 72% security failure rate** for AI-generated code (Veracode 2025). Model size doesn't help.

### Ruby on Rails Specifically

Rails is one of the better frameworks for agentic coding:
- Top 3 in the mame benchmark (speed, cost, stability). Zero failures across all trials.
- Convention over configuration gives LLMs strong structural priors
- Ruby's expressiveness produces compact output (fewer tokens, lower cost)
- **Risk:** LLMs sometimes produce "Python-ish Ruby" — enforce idioms in CLAUDE.md
- **Risk:** Dynamic metaprogramming (`method_missing`, DSLs) confuses models
- **Tooling:** `rails-ai-context` gem auto-generates CLAUDE.md from app structure. `palkan/skills` has a Layered Rails architectural review skill.

### Skill Design Implications

- Skills targeting Python will work best out-of-the-box
- Skills targeting TypeScript/Java/Rust benefit from compiler-as-verifier patterns
- Skills for JavaScript/Ruby need extra runtime verification steps
- Framework-specific context (like AGENTS.md for Next.js) dramatically outperforms generic approaches
- Include a "gold standard file" — a reference implementation demonstrating all conventions (Stack Overflow pattern)

---

## Part 6: Anti-Pattern Reference

| Anti-Pattern | What Happens | Prevention |
|---|---|---|
| **Circular validation** | AI writes tests from implementation; both contain same bug | Test writer must not see implementation |
| **Fire-and-forget** | Vague prompt → compound errors | Structured specs, bounded tasks, checkpoints |
| **Kitchen-sink session** | Mixing unrelated tasks → context pollution | `/clear` between tasks |
| **Correction loops** | Repeated fixes in polluted context | After 2 failures, `/clear` and restart |
| **Over-specified CLAUDE.md** | Too many instructions → ignored | <200 lines, modular via `@import` |
| **Layer-by-layer decomposition** | Untestable partial implementations | Capability-ordered decomposition |
| **Same-session self-review** | Confirmation bias | Selective context transfer, cross-model |
| **Test theater** | High coverage, tests verify nothing | Tests must fail when assertions removed |
| **Agent self-orchestration** | Agents skip steps, create loops | Deterministic orchestration |
| **Premature claims** | "It should work" without evidence | Verification gate with command output |
| **Abstraction addiction** | Unnecessary patterns and layers | Constrain in rules |
| **Deleting failing tests** | Agent removes tests vs fixing code | Coverage regression gates |
| **Over-specifying small tasks** | 4 user stories for a bug fix | Proportional ceremony |
| **Quality inversion** | AI fixes shallow bugs, introduces deep architectural flaws | Security review, architecture review |
| **Requirement drift** | Agents miss explicit requirements | Verification skill cross-references original request |

---

## Part 7: Key Data Points

Annotated with evidence quality. Use these to calibrate expectations, not as hard rules.

| Metric | Value | Quality | Source |
|---|---|---|---|
| Bug density without guardrails | +35-40% | Survey data | Qodo 2025 |
| Devs reporting quality improvement with review | 81% | Self-reported survey | Qodo 2025 |
| Context degradation threshold | ~60% utilization | Practitioner observation | Multiple |
| AI code security failure rate | 45% | Empirical (100+ models) | Veracode 2025 |
| Compound accuracy (85%/step, 10 steps) | ~20% e2e | Back-of-envelope calc | Illustrative |
| CLAUDE.md instruction budget | ~150-200 lines | Practitioner observation | Not rigorously measured |
| Code Health for reliable AI | 9.5/10 | Peer-reviewed | CodeScene |
| Atomic vs bundled task cost | 59% less | Single anecdote | One blog post |
| METR: experienced devs + AI | 19% slower | RCT, n=16 | METR 2025 (Claude 3.5 era) |
| Multi-file sessions (2026) | 78% | First-party data | Anthropic Q1 2026 |
| Stripe agent PRs/week | 1,000+ | First-party blog | Stripe Engineering |
| Python vs TypeScript resolve rate | 52% vs 2-49% | Benchmarks vary | Multi-SWE-bench (Claude 3.7) |
| Always-loaded skill text ceiling | ~2,500 tokens | Source leak analysis | March 2026 |
| Prompt cache cost savings | 5x | Source leak analysis | March 2026 |

---

## Part 8: Pre-Built Ecosystem

Before building skills from scratch, check what exists:

| Source | What | Credibility |
|---|---|---|
| **anthropics/skills** (110K stars) | 17 official skills — reference implementation | Canonical |
| **Vercel plugin** (this session) | 47+ skills for Next.js, AI SDK, etc. | Production-quality |
| **Trail of Bits claude-code-config** (1.8K stars) | Security harness — deny lists, sandboxing, hooks | Best security config |
| **VoltAgent/awesome-agent-skills** (14K stars) | 1,060+ curated skills from vendors | Good curation |
| **skills.sh** | Registry/package manager for skills | Emerging standard |
| **palkan/skills** | Layered Rails architectural review | High-quality, Rails-specific |
| **Stripe, Cloudflare, Netlify, HashiCorp** | Official vendor skills via VoltAgent | Maintained by product teams |

---

## Part 9: Self-Improving Skills

**Maturity reality:** Most teams are at level 1-2 (manual or human-triggered). Fully automated self-improvement is research-only.

**The spectrum:**
1. **Manual** — Human spots mistake, updates CLAUDE.md. Where most teams are.
2. **Human-triggered** — Human tells agent: "Reflect on this mistake. Abstract the learning. Write it to CLAUDE.md." Boris Cherny's "compounding engineering" pattern.
3. **Semi-automated** — claude-reflect captures corrections via regex + semantic analysis, presents for human review before syncing to config files. pro-workflow uses SQLite + FTS5 for persistent learning storage.
4. **Fully automated** — SICA (ICLR 2025) edits its own source code. Not production-ready.

**The unsolved problem:** Trusting a flawed agent to write its own corrections. Human review gate on rule changes is currently required.

**Stop hooks as verification:** A prompt-based stop hook that cross-references agent output against input requirements catches "agent thinks it's done but missed something." This would have caught our own failure (7 agents missing an explicit user request).

---

## Part 10: Skill Writing Craft

Synthesized from analysis of Anthropic's official skills (doc 20), Vercel's most-installed skills (doc 21), Trail of Bits' security harness (doc 22), top community skills including Superpowers, palkan, HashiCorp, BMAD (doc 23), published skill writing guides (doc 24), and practitioner skills (doc 25).

### The Description Field Is Make-or-Break

The description is the trigger mechanism — it determines whether the skill activates at all. Anthropic's skill-creator calls descriptions "pushy" by design because Claude under-triggers by default.

**Critical rule:** Descriptions must contain only triggering conditions ("Use when..."), never process summaries. Descriptions that summarize the workflow cause Claude to follow the summary and skip the skill body entirely.

- 50-150 words
- Enumerate specific use cases and negative triggers ("DO NOT TRIGGER when...")
- Use task verbs ("writing", "reviewing"), not technology nouns

### Three Skill Archetypes

Match the format to the nature of the knowledge (from Vercel's patterns):

1. **Reference skills** — Neutral docs, lazy-loaded rules. Hub SKILL.md (~60 lines) as routing index, detailed rules in `references/`. Best for: framework best practices, API guides. (Example: Remotion, 117K installs, 60-line hub)

2. **Process skills** — Imperative checklists with phase gates. Monolithic SKILL.md with sequential phases. Best for: workflows like define → design → execute. (Example: Anthropic's mcp-builder)

3. **Discipline skills** — Adversarial prose with rationalization tables and red flag lists. Best for: enforcing practices the agent will resist (TDD, review rigor). (Example: Superpowers' TDD skill with "Iron Laws" and escape-attempt detection)

### Prose Style

Patterns that appear across ALL high-quality skills:

- **Imperative, no hedging.** "State findings as recommendations, not questions" — not "you might want to consider stating findings as recommendations."
- **Explain WHY, not just WHAT.** Anthropic's central thesis: Claude generalizes better from reasoning than from commands. One sentence of WHY beats three sentences of WHAT.
- **Show, don't tell.** Every rule should have an Incorrect/Correct code pair (Vercel pattern). Examples beat explanations. One snippet beats three paragraphs of description.
- **Quantify impact.** "2-10x improvement" or "200-800ms savings" — not "significant improvement" (Trail of Bits explicitly bans inflated language: "avoid: critical, crucial, essential, significant, comprehensive, robust, elegant").
- **Encode judgment, not just knowledge.** The value isn't listing functions — it's decision trees that route to the right approach. "When there's a clear best approach, present it directly. Don't manufacture alternatives."

### Progressive Disclosure in Practice

The three-layer model (metadata → instructions → resources) is the key architectural insight. In practice:

- **SKILL.md body:** Decision trees, phase gates, key rules. Under 500 lines.
- **references/:** Detailed documentation, API references, convention libraries. Loaded on demand.
- **scripts/:** Deterministic operations as executable black boxes. The agent runs them without understanding internals.
- **assets/:** Templates, boilerplate, reference implementations. The "gold standard file" pattern — a reference implementation demonstrating all conventions.

**Token budget reality:** Always-loaded skill text ceiling is ~2,500 tokens. SKILL.md body loaded on activation should stay under 5,000 tokens. Reference files have no practical limit since they're loaded on demand.

### Adversarial Design

The Superpowers collection pioneered the most sophisticated technique for skills that enforce discipline:

- **Rationalization tables** — Lists of thoughts the agent might have that are red flags: "I can skip this step because...", "This is just a simple change...", "The user probably doesn't need..."
- **Iron Laws** — Non-negotiable rules that cannot be overridden regardless of context
- **Red flag lists** — Specific phrasings that indicate the agent is about to cut corners
- **The phrase "your human partner"** — Creates behavioral constraints by framing the relationship

### Hook Error Messages as Teaching

Trail of Bits' pattern: hook block messages should redirect, not just block. "BLOCKED: Use `trash` instead of `rm -rf`" teaches the agent the correct alternative in the same moment it prevents the mistake. One-shot learning via enforcement.

### Testing Skills

From Anthropic's platform docs and Stack Overflow:

- **Evaluation-driven development** — Write test cases for the skill itself before writing the skill
- **Two-Claude testing** — One Claude instance uses the skill, another evaluates whether the output is good
- **Adversarial testing** — "Approach your instructions in the most bad-faith manner as possible" to find loopholes
- **The error-driven flywheel** — Only add rules when you observe failures. Don't anticipate problems.

### What Great Skills Do NOT Include

- Installation instructions or meta-commentary about the skill itself
- Things Claude already knows (standard language conventions, common patterns)
- Style rules that a linter can enforce (offload to hooks)
- Detailed API documentation (link to docs or put in `references/`)
- Hedging language ("you might want to", "consider perhaps")

---

## Part 11: Reading Order for Skill Authors

An agent authoring skills should read documents in this order:

1. **This document (00)** — principles, pipeline, skill set, craft guide
2. **08-agent-skills-standard.md** — MUST READ. The actual Agent Skills spec.
3. **24-skill-writing-craft.md** — MUST READ. The meta-skill: how to write skills well.
4. **20-anthropic-skill-analysis.md** — How Anthropic writes skills (the canonical reference)
5. **21-vercel-skill-analysis.md** — How Vercel writes skills (the most-installed)
6. **22-trailofbits-analysis.md** — How Trail of Bits writes security configs
7. **23-community-skill-analysis.md** — Patterns from the best community skills
8. **09-source-leak-insights.md** — Sub-agent models, prompt caching, memory architecture
9. **10-anthropic-official-docs.md** — Hooks, subagents, plugins, auto mode
10. **17-red-team-report.md** — Counterarguments and edge cases for each principle
11. **18-prebuilt-skills-ecosystem.md** — Check what already exists before building
12. **25-practitioner-skill-analysis.md** — Analysis of a working define/design/execute pipeline

Language/framework specific (read if relevant):
- **11-language-framework-impact.md** — Performance gaps across languages
- **14-ruby-rails-impact.md** — Ruby on Rails specifically

For specific skill types:
- **13-self-improving-skills.md** — For the retrospective/self-improvement skill
- **19-ralph-loop.md** — For autonomous iteration patterns

Supporting research (01-07, 12, 15, 16) provides evidence behind the principles but is not required for skill authoring.

---

## Part 11: Sources and Supporting Documents

| Doc | Contents |
|---|---|
| `01-tool-landscape.md` | Survey of 12 tools, comparison matrix |
| `02-spec-driven-dev.md` | Kiro, Antigravity, Spec Kit, BMAD, AGENTS.md, context engineering |
| `03-implementation.md` | Decomposition, multi-agent patterns, git workflow, context management |
| `04-quality-assurance.md` | TDD, code review, verification, guard rails, trust spectrum |
| `05-community-wisdom.md` | Practitioner opinions (Willison, Beck, Osmani, Tornhill, Cherny) |
| `06-anti-patterns.md` | Vibe coding failures, production incidents, security |
| `07-enterprise-vs-startup.md` | FAANG, startup, OSS, consulting, Codex approaches |
| `08-agent-skills-standard.md` | **The Agent Skills spec, adoption, ecosystem** |
| `09-source-leak-insights.md` | **Sub-agent models, prompt caching, memory architecture, KAIROS** |
| `10-anthropic-official-docs.md` | **Hooks (26 events, 4 types), subagents, plugins, auto mode** |
| `11-language-framework-impact.md` | **Language performance gaps, type systems, codebase health** |
| `12-missed-practitioners.md` | Gold standard file pattern, adversarial testing, MIT Missing Semester |
| `13-self-improving-skills.md` | **Self-correction spectrum, claude-reflect, stop hooks** |
| `14-ruby-rails-impact.md` | **Ruby/Rails benchmark data, tooling, practitioner experiences** |
| `15-verification-report.md` | Quality audit of all research (7 failures found and fixed in v2) |
| `16-singhcoder-repo.md` | The SinghCoder/claude-code fork — a porting scaffold, not a working tool |
| `17-red-team-report.md` | **Counterarguments to every principle — 4 STRONG, 4 MODERATE** |
| `18-prebuilt-skills-ecosystem.md` | **Pre-built skills from Anthropic, Vercel, Trail of Bits, vendors** |
| `19-ralph-loop.md` | **Autonomous iteration pattern — fresh context per loop** |
| `20-anthropic-skill-analysis.md` | **How Anthropic writes skills — the canonical style** |
| `21-vercel-skill-analysis.md` | **How Vercel writes skills — three archetypes, build pipelines** |
| `22-trailofbits-analysis.md` | **Trail of Bits security harness — hooks as teaching, anti-LLM-speak** |
| `23-community-skill-analysis.md` | **Community skills — Superpowers adversarial design, palkan, HashiCorp** |
| `24-skill-writing-craft.md` | **The meta-skill — 10 principles of writing skills well** |
| `25-practitioner-skill-analysis.md` | **Analysis of a working define/design/execute pipeline** |

---

*v3 of this spec. Incorporates craft analysis from docs 20-25. All verifier failures (doc 15) have been addressed. Source-checked corrections applied (Code Health 9.5, Stripe 1,000+, DORA/TDD attribution). Red team nuance integrated into principles. Evidence quality annotated throughout.*
