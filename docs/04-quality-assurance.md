# Quality Assurance for Agentic Coding

Research into how top engineering teams maintain code quality when using AI coding agents. Focused on real practices from real teams as of early 2026.

---

## The Problem in Numbers

The data is unambiguous: AI-generated code ships faster but breaks more often without guardrails.

- AI-generated code introduces **1.7x more issues** overall vs human-written code (Qodo 2025 State of AI Code Quality)
- **1.75x more logic/correctness errors** in AI output -- the kind that cause production incidents
- **29-45% of AI-generated code contains security vulnerabilities** (Veracode 2025)
- Teams using AI assistants **without quality guardrails report 35-40% increase in bug density** within 6 months
- **19.7% of recommended packages are fabricated** (hallucinated dependencies)
- PRs are ~18% larger with AI adoption; incidents per PR up ~24%, change failure rate up ~30% (Addy Osmani)

The flip side: teams that pair AI with rigorous review see **81% report quality improvements** vs 55% without review (Qodo). The tool is not the problem. Absence of discipline is.

### The Dunning-Kruger Effect in AI Coding

The Qodo report surfaced a dangerous pattern: **junior developers (<2 years experience) report the lowest quality improvements (51.9%) from AI but the highest confidence (60.2%) in shipping AI code without review.** Senior developers (10+ years) report the highest quality benefits (68.2%) but the most caution -- only 25.8% would ship without review. The people who benefit most from AI are the ones least likely to trust it blindly.

### The Vibe Coding Cautionary Tales

"Vibe coding" -- describing what you want in natural language, accepting whatever AI generates, shipping without review -- has produced real production disasters:

- **Amazon March 2026**: AI-assisted deployment caused 6-hour shutdown, ~6.3M lost orders
- **Replit agent wiped SaaStr's production database** without permission (reversed via checkpoints)
- **Claude Code ran `terraform destroy`** on 2.5 years of production infrastructure
- **Tea App (July 2025)**: Exposed 72,000 images including 13,000 government IDs -- Firebase left on default settings
- **Moltbook**: 1.5M authentication tokens exposed
- A December 2025 study found 69 vulnerabilities across 15 apps built by 5 major AI coding tools. Every single app lacked CSRF protection. Zero apps set security headers.

---

## 1. Code Review of AI-Generated Code

### The Fundamental Rule

Simon Willison, in his *Agentic Engineering Patterns* guide (Feb 2026): **"Don't file pull requests with code you haven't reviewed yourself... You are responsible for all code you push, and you are responsible for not wasting your colleagues' time."**

Addy Osmani frames the new reviewer role: AI reviewers catch 70-80% of low-hanging fruit, freeing humans to focus on architecture, business logic, roadmap alignment, and institutional context AI cannot grasp. The human reviewer becomes an **editor and architect**, not a line-by-line proofreader.

### Self-Review Patterns (Agent Reviews Its Own Code)

The most effective pattern emerging is the **Writer/Reviewer separation**: one agent writes code, a fresh agent (in a clean context) reviews it. This avoids confirmation bias -- the writer is naturally biased toward its own output.

**Claude Code Review** (launched March 2026) implements this at scale:
- Dispatches **parallel specialized agents** on every PR
- Each agent targets a different class of issue (logic errors, boundary conditions, API misuse, auth flaws, convention compliance)
- A **verification step attempts to disprove each finding** before posting -- a deliberate false-positive filter
- Flags problems in 84% of changes over 1,000 lines
- Averages 7.5 issues per change with **false positive rate below 1%**

The HAMY Labs approach runs **9 parallel subagents**, each focused on a specific quality dimension: linting, code review, security review, style review, etc. Each finding gets a 0-100 confidence score; only high-confidence issues (default threshold: 80) get reported.

### Hybrid Human-AI Review

The emerging consensus workflow:
1. **Agent writes code** with tests
2. **Agent self-reviews** in fresh context (catches mechanical issues)
3. **Automated gates** run (lint, typecheck, test suite)
4. **Human reviews** for intent, architecture, domain correctness
5. **Human owns the merge decision** -- always

### The REVIEW.md Pattern

Teams customize AI review behavior through two files:
- **CLAUDE.md**: General project context and coding conventions
- **REVIEW.md**: Specific review scope and priorities (what to look for, what to ignore)

This gives the AI reviewer project-specific judgment without cluttering the general instruction file.

---

## 2. Testing Strategies with Agents

### TDD as the Premier Quality Strategy

TDD has emerged as the single most effective quality strategy for agentic coding. The consensus across multiple sources is striking.

**Why TDD works so well with AI** (Codemanship, Jan 2026):
- TDD forces working **one problem at a time** -- prevents the "jumbled mess" of generating too much at once
- Tests provide a **binary success signal** -- the clearest goal you can give an AI
- Rapid feedback loops catch issues **before they compound**
- Tests **pin down the meaning of requirements**, producing more accurate output from the model
- DORA data shows elite-performing teams are "pretty much all doing TDD"

**Nathan Fox's approach** (Aug 2025): Embed TDD directly in CLAUDE.md so it's the default behavior:
1. Generate code and modify types
2. Create functions with stubbed return values
3. Create unit tests
4. Implement functions progressively
5. Run unit tests to verify

**The forcing function insight**: You cannot prompt your way into TDD discipline. You need **structural enforcement** -- hooks, skills, and phase gates that block progression until each step completes. The alexop.dev implementation uses explicit phase gates: Red (write failing test) -> Green (minimal implementation) -> Refactor (clean up) -- the agent cannot skip phases.

**Context isolation for honest tests**: Subagent-based TDD separates the test writer from the implementer. The test-writing agent **cannot see implementation plans**, so tests reflect actual requirements rather than anticipated code structure. This is critical -- when the same agent writes both tests and implementation, it unconsciously writes tests that match its implementation rather than the spec.

### Test-After Patterns

When TDD is not feasible (legacy code, exploratory work):
- Agent implements feature
- Agent writes tests in a **fresh context** (avoids writing tests that just confirm what it already wrote)
- Tests must pass AND fail when key assertions are commented out (proving they actually test something)
- Human reviews test quality specifically -- AI-generated tests are the most dangerous form of "looks right but tests nothing"

### How Agents Handle Test Failures

The agentic advantage: when tests fail, the agent can **iterate autonomously** -- read the error, modify code, re-run tests -- in a tight loop. This is where AI excels. The key constraint: set a **maximum iteration count** (typically 3-5 attempts). If the agent cannot fix the test after N tries, it should stop and report rather than thrashing.

---

## 3. Verification and Validation

### Beyond "It Compiles"

AI-generated code often "works" superficially but contains subtle bugs. The verification stack:

1. **Type checking** -- catches the most mechanical errors (mismatched types, missing fields)
2. **Linting** -- enforces conventions AI frequently violates (import ordering, naming, etc.)
3. **Unit tests** -- verifies behavior at the function level
4. **Integration tests** -- verifies components work together
5. **Browser/E2E testing** -- verifies the actual user experience

### Agentic Browser Testing

Emerging tools for agent-driven browser verification:
- **BrowserStack AI** (2025): Autonomous agents generate test cases, execute across real devices, adapt to UI changes
- **Testsigma Atto** (2025): AI agents generate, adapt, and manage test execution
- Claude Code can drive browser automation via MCP tools to verify its own output visually

The pattern: after implementing a UI change, the agent launches the dev server, navigates to the affected page, and visually/programmatically verifies the change works. This catches the class of bugs where code is syntactically correct but visually broken.

### The Verification-Before-Completion Pattern

A critical anti-pattern: agents claiming work is "done" without running verification. The fix is structural:
- Before any commit, the agent MUST run: `typecheck`, `lint`, `test suite`
- Before claiming completion, the agent MUST show **evidence** (actual command output) not just assertions
- "It should work" is never acceptable -- "here's the passing test output" is the minimum bar

---

## 4. Guard Rails and Constraints

### The Layered Defense Model

Guardrails work as a layered stack where each layer catches different failure modes:

| Layer | Mechanism | Catches |
|-------|-----------|---------|
| **CLAUDE.md** | Instructions | Convention violations, architectural drift |
| **Hooks (PreToolUse)** | Deterministic blocks | Dangerous commands, protected files |
| **Hooks (PostToolUse)** | Auto-formatting | Style issues, import ordering |
| **Pre-commit hooks** | Git-level gates | Lint errors, type errors, test failures |
| **CI pipeline** | Remote verification | Integration failures, security scans |
| **AI code review** | Semantic analysis | Logic errors, missed edge cases |
| **Human review** | Judgment | Intent, architecture, domain correctness |

### Claude Code Hooks

Hooks are deterministic -- they fire every time, unlike prompt instructions which AI may ignore. Three handler types:

1. **Command hooks**: Run shell commands (formatters, linters). Fast, reliable.
2. **Prompt hooks**: Inject additional context/instructions at decision points. Semantic guidance.
3. **Agent hooks**: Spawn subagents for deep analysis. Expensive but thorough.

**PreToolUse** is the only hook that can **block** actions. Use for:
- Preventing writes to protected files
- Blocking dangerous commands (`rm -rf`, `terraform destroy`)
- Enforcing mandatory review before commits

**PostToolUse** hooks for automatic cleanup:
- Run formatter after every file edit
- Run linter and surface issues immediately
- Trigger test suite on save

### Trail of Bits Configuration

Trail of Bits published their Claude Code security configuration (open source), which sets:
- Code quality hard limits (function length, complexity, line width)
- Language-specific toolchains (ruff for Python, oxlint for Node, clippy for Rust)
- Testing methodology requirements
- Workflow conventions

Their philosophy: **"guardrails, not walls"** -- hooks are structured interventions at decision points, not a security boundary.

### Never Send an LLM to Do a Linter's Job

Critical insight from multiple sources: **do not put style rules in CLAUDE.md that a linter can enforce deterministically.** LLMs are slow, expensive, and unreliable for style enforcement. Use eslint/prettier/ruff/rubocop as PostToolUse hooks. Reserve CLAUDE.md for semantic guidance that tools cannot enforce (architectural patterns, domain conventions, design philosophy).

### CLAUDE.md Budget

There is roughly a **150-200 instruction budget** before compliance drops off. The system prompt uses ~50 of those. Keep CLAUDE.md under 200 lines per file. Use **progressive disclosure** -- task-specific instructions in separate files that are only referenced when relevant, not loaded every session.

---

## 5. The Trust Spectrum

### Anthropic's Own Data on Autonomy

Anthropic's research on Claude Code usage patterns:
- Newer users employ full auto-approve ~20% of the time
- By 750 sessions, this increases to **over 40%**
- Trust builds gradually through accumulated experience
- **Experienced users interrupt more often, not less** -- they have better instincts for when intervention is needed

### The Intern Model

Developer communities call the most effective approach the **"intern model"**: treat the AI as a capable junior developer who still requires supervision. This is the mode where experienced developers report the best results. The key behavior: **let the agent work autonomously, but intervene when something looks wrong** -- active monitoring, not passive acceptance.

### What to Delegate vs. Supervise

**High autonomy (let the agent work):**
- Boilerplate generation (models, migrations, CRUD)
- Test writing (especially from clear specs)
- Refactoring with existing test coverage
- Bug fixes with clear reproduction steps
- Code formatting and cleanup

**Require human checkpoints:**
- Authentication and authorization logic
- Payment processing
- Database schema design
- API design (public contracts)
- Security-sensitive code
- Architectural decisions
- Anything without test coverage to verify against

**Never delegate:**
- Production deployments without human approval
- Destructive operations (database drops, terraform destroy)
- Security configuration
- Dependency version bumps without review (supply chain risk)

### The 30/70 Split

Addy Osmani's framing: developers now spend ~30% of their time on the uniquely human parts (intent, architecture, judgment) and ~70% is AI-assisted. But that 30% is where all the quality decisions happen. The AI handles volume; the human handles judgment.

---

## 6. Failure Modes and Recovery

### Common AI Code Failure Modes

Research identifies three primary hallucination categories:

1. **Task Requirement Conflicts**: Code that looks right but doesn't match what was actually asked for
2. **Factual Knowledge Conflicts**: Using APIs that don't exist, wrong function signatures, fabricated packages
3. **Project Context Conflicts**: Ignoring existing patterns, duplicating functionality, breaking conventions

Specific failure patterns:
- **Plausible but wrong logic**: Code that passes casual review but has subtle edge-case bugs
- **Over-engineering**: Adding unnecessary abstractions, patterns, and indirection
- **Ignoring existing code**: Reimplementing something that already exists in the codebase
- **Stale knowledge**: Using deprecated APIs or outdated patterns
- **Test theater**: Tests that pass but don't actually verify anything meaningful
- **Configuration drift**: Changing settings files in ways that break other parts of the system

### Prevention Strategies

**Context is king**: When an AI agent has access to the right code context -- local files, architectural patterns, team conventions -- hallucination rates plummet. The best prevention is better context, not better prompting.

**Specific prevention techniques:**
- Ask for citations or API references where possible
- Run the same query multiple times and compare outputs (if results differ, investigate)
- Use a "verifier" agent to check the "writer" agent's output
- Require the agent to explain *why* it made each decision (forces reasoning, catches hand-waving)
- Use type systems aggressively -- they catch the largest class of mechanical errors

### Recovery and Rollback

**Git is the recovery mechanism.** The most reliable rollback strategies:

1. **Frequent commits**: Commit after each verified step, not at the end. Each commit is a checkpoint.
2. **Git worktrees for isolation**: Each agent works in its own worktree/branch. If it goes off the rails, discard the branch. No impact on main.
3. **The checkpoint pattern**: Before the agent starts work, note the current commit hash. If things go wrong, `git reset --hard <checkpoint>`.
4. **Branch-per-task**: Each agent task gets its own branch. Merge only after review. Failed tasks are simply abandoned branches.

**What git cannot recover:**
- External side effects (API calls, database mutations, cloud resource changes)
- Destructive operations that bypass git (direct file system operations outside the repo)
- Terraform/infrastructure changes

For these, the answer is **prevention not recovery**: PreToolUse hooks that block dangerous commands, sandboxing that limits network access, and mandatory human approval for any destructive external operation.

### Git Worktrees for Parallel Safety

Git worktrees have become the standard isolation mechanism for multi-agent development:
- Each agent gets its own worktree (isolated working directory + branch)
- No file-level conflicts between parallel agents
- IDE support improving (VS Code added worktree support July 2025)
- **Limitation**: Worktrees share the same database, Docker daemon, and cache -- race conditions possible for stateful resources

---

## 7. Emerging Patterns and Principles

### The Plan-Execute-Verify Loop

The most reliable agentic workflow follows three phases:
1. **Plan**: Agent proposes approach, human validates before any code is written
2. **Execute**: Agent implements with TDD, running tests at each step
3. **Verify**: Fresh agent reviews in clean context; human reviews for intent

Teams report this reduces correction iterations by ~60% on complex tasks.

### Configuration as Code

The best teams treat their AI configuration like infrastructure code:
- **CLAUDE.md** in version control, reviewed in PRs
- **Hooks** defined in `.claude/settings.json`, committed to repo
- **Custom commands** in `.claude/commands/`, shared across team
- **REVIEW.md** for review-specific guidance

### The Experienced-Developer Paradox

A controlled study found experienced open-source developers were **19% slower** when using AI coding tools -- despite predicting they would be 24% faster, and still *believing afterward* they had been 20% faster. The lesson: subjective experience of AI productivity can be misleading. Measure actual outcomes (bug rates, cycle time, incident frequency), not feelings.

### Multi-Agent Architecture for Quality

The trend is toward specialized agents rather than one generalist:
- **Writer agent**: Implements code
- **Test agent**: Writes tests (isolated from implementation)
- **Review agent**: Reviews in fresh context
- **Security agent**: Focused security analysis
- **Style agent**: Convention compliance

Each agent has limited scope and clear success criteria. This mirrors how human teams work -- separation of concerns applied to AI.

---

## Key Takeaways for Skill Design

1. **TDD is not optional** -- it is the single highest-leverage quality practice for agentic coding. Skills should enforce the Red-Green-Refactor cycle structurally, not just suggest it.

2. **Deterministic gates beat probabilistic instructions** -- hooks and linters always fire; CLAUDE.md instructions sometimes get ignored. Use deterministic enforcement for everything that can be automated.

3. **Fresh context for review** -- the agent that wrote the code should never be the only reviewer. Spawn a subagent or use a separate session for review.

4. **Evidence before assertions** -- require actual command output (test results, lint results) before accepting claims of completion. "It should work" is never sufficient.

5. **Layer defenses** -- no single mechanism catches everything. Stack type checking + linting + tests + AI review + human review.

6. **Isolate with git worktrees** -- for parallel agent work, worktrees provide filesystem isolation. For sequential work, frequent commits provide rollback checkpoints.

7. **Protect the dangerous operations** -- PreToolUse hooks must block destructive commands. Human approval gates for anything that touches production, infrastructure, or security.

8. **Keep CLAUDE.md lean** -- under 200 lines, focused on things tools cannot enforce. Offload style rules to linters, formatting to formatters.

9. **Trust builds with experience** -- the intern model works. Let agents work autonomously on low-risk tasks; require checkpoints on high-risk ones. Adjust the boundary as you learn the agent's strengths and weaknesses.

10. **Measure outcomes, not feelings** -- track bug density, incident rates, and cycle time. Subjective assessments of AI productivity are unreliable.

---

## Sources

- [Qodo: State of AI Code Quality 2025](https://www.qodo.ai/reports/state-of-ai-code-quality/)
- [Addy Osmani: Code Review in the Age of AI](https://addyo.substack.com/p/code-review-in-the-age-of-ai)
- [Addy Osmani: Treat AI-Generated Code as a Draft](https://addyo.substack.com/p/treat-ai-generated-code-as-a-draft)
- [Simon Willison: Agentic Engineering Patterns](https://simonwillison.net/2026/Feb/23/agentic-engineering-patterns/)
- [Codemanship: Why TDD Works Well in AI-Assisted Programming](https://codemanship.wordpress.com/2026/01/09/why-does-test-driven-development-work-so-well-in-ai-assisted-programming/)
- [Nathan Fox: Taming GenAI Agents with TDD](https://www.nathanfox.net/p/taming-genai-agents-like-claude-code)
- [alexop.dev: Forcing Claude Code to TDD](https://alexop.dev/posts/custom-tdd-workflow-claude-code-vue/)
- [Steve Kinney: Test-Driven Development with Claude Code](https://stevekinney.com/courses/ai-development/test-driven-development-with-claude)
- [The New Stack: Claude Code and TDD](https://thenewstack.io/claude-code-and-the-art-of-test-driven-development/)
- [Trail of Bits: Claude Code Config](https://github.com/trailofbits/claude-code-config)
- [Dwarves Foundation: Claude Guardrails](https://github.com/dwarvesf/claude-guardrails)
- [Claude Code Docs: Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code Docs: Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Docs: Code Review](https://code.claude.com/docs/en/code-review)
- [Anthropic: Measuring Agent Autonomy](https://www.anthropic.com/research/measuring-agent-autonomy)
- [HAMY Labs: 9 Parallel AI Agents That Review My Code](https://hamy.xyz/blog/2026-02_code-reviews-claude-subagents)
- [Guardrails for Agentic Coding (jvaneyck)](https://jvaneyck.wordpress.com/2026/02/22/guardrails-for-agentic-coding-how-to-move-up-the-ladder-without-lowering-your-bar/)
- [Quality Gates in the Age of Agentic Coding (helio)](https://blog.heliomedeiros.com/posts/2025-07-18-quality-gates-agentic-coding/)
- [Stack Overflow: Are Bugs Inevitable with AI Coding Agents?](https://stackoverflow.blog/2026/01/28/are-bugs-and-incidents-inevitable-with-ai-coding-agents/)
- [Vibe Coding Failures: Documented Incidents](https://crackr.dev/vibe-coding-failures)
- [Amazon Vibe Coding Failures](https://www.getautonoma.com/blog/amazon-vibe-coding-lessons)
- [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [AI Coding Spectrum: 6 Levels of Assistance](https://eclipsesource.com/blogs/2025/06/26/ai-coding-spectrum-levels-of-assistance/)
- [How Reversible is an Agentic Mistake? (IT Brew)](https://www.itbrew.com/stories/2026/03/06/how-reversible-is-an-agentic-mistake)
- [arXiv: Survey of Bugs in AI-Generated Code](https://arxiv.org/html/2512.05239v1)
- [InfoWorld: How to Keep AI Hallucinations Out of Your Code](https://www.infoworld.com/article/3822251/how-to-keep-ai-hallucinations-out-of-your-code.html)
