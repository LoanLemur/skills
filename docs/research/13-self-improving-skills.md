# Self-Improving Skills: Automated Feedback Loops for Agentic Coding

## Maturity Assessment

This is an **emerging practice area**. Most teams still update their agent configurations manually. Fully automated self-improvement loops exist primarily in research papers and experimental tools, not production workflows. However, several concrete patterns are being tried in the wild, and the direction of travel is clear.

The spectrum runs from:
1. **Manual** (human spots mistake, human updates CLAUDE.md) -- where most teams are today
2. **Human-triggered** (human spots mistake, tells agent to update its own rules) -- Boris Cherny's pattern
3. **Semi-automated** (hooks detect patterns, surface proposed changes for human review) -- claude-reflect, pro-workflow
4. **Fully automated** (agent detects failure, updates own configuration, verifies improvement) -- mostly theoretical for coding agents; demonstrated in research (SICA)

---

## 1. The Manual Baseline: Compounding Engineering

Boris Cherny (creator of Claude Code) established the foundational pattern: "Anytime we see Claude do something incorrectly, we add it to CLAUDE.md." The team contributes multiple times weekly, treats it as a living document checked into git, and calls the practice "compounding engineering" -- each correction becomes permanent context.

**Key practices from the Claude Code team:**
- Tag `@.claude` on PR reviews to trigger the GitHub Action to update CLAUDE.md automatically
- Ruthlessly edit CLAUDE.md over time; measure whether mistake rates drop
- Create notes directories for every task/project, updated after each PR
- Check slash commands into `.claude/commands/` for repeated workflows

This is manual in the sense that a human must notice the mistake. But the *writing* of the rule is delegated to Claude, which "is eerily good at writing rules for itself."

**Source:** [How Boris Uses Claude Code](https://howborisusesclaudecode.com/), [10 Tips from Inside the Claude Code Team](https://paddo.dev/blog/claude-code-team-tips/)

---

## 2. Human-Triggered Self-Correction

The next step: instead of manually writing the rule, tell Claude to reflect and write it.

### The Reflection Prompt

A single instruction -- "Reflect on this mistake. Abstract and generalize the learning. Write it to CLAUDE.md." -- triggers a three-step process:

1. **Reflect**: Analyze what went wrong while context is still in memory
2. **Abstract**: Extract the general principle (e.g., "don't patch logger" becomes "avoid patching infrastructure")
3. **Generalize**: Create a reusable decision framework

Rules follow a strict format: lead with NEVER/ALWAYS directives, explain the problem before the solution, keep to 1-3 bullets, include concrete commands. A META section in CLAUDE.md teaches Claude *how to write rules*, creating self-regulating quality control.

**Source:** [Self-Improving AI: One Prompt That Makes Claude Learn From Every Mistake](https://dev.to/aviad_rozenhek_cba37e0660/self-improving-ai-one-prompt-that-makes-claude-learn-from-every-mistake-16ek)

### Eric Ma's Runbook/Postmortem Analogy

Eric Ma frames agent improvement as operational practice: "If the model weights are not changing mid-week, improvement has to come from the environment you wrap around the agent." Two levers:

1. **AGENTS.md as repository memory** -- navigation maps reduce exploration time from 40s to 2s; local norms prevent repeated corrections
2. **Skills as reusable playbooks** -- multi-step procedures that execute consistently across sessions

When corrections occur, agents update AGENTS.md themselves, creating durable feedback.

**Source:** [How to Build Self-Improving Coding Agents - Part 1](https://ericmjl.github.io/blog/2026/1/17/how-to-build-self-improving-coding-agents-part-1/)

---

## 3. Semi-Automated Systems (Tools in the Wild)

### claude-reflect

A skill that captures corrections and syncs them to configuration files. Architecture:

- **Hybrid detection**: Regex patterns identify correction phrases ("no, use X", "that's wrong") + semantic AI analysis validates and extracts actionable insights
- **Human review gate**: `/reflect` presents captured learnings with confidence scores; users approve, edit, or skip before sync
- **Multi-target propagation**: Approved learnings go to `~/.claude/CLAUDE.md` (global), `./CLAUDE.md` (project), `.claude/commands/*.md` (skills), and `AGENTS.md` (cross-tool)
- **Skill discovery**: `/reflect-skills` analyzes session history semantically, recognizing intent similarity across different wordings. When patterns recur (e.g., "review productivity" phrased 4 different ways), proposes a reusable skill

Corrections during skill execution strengthen the skill's definition over time, creating a feedback loop.

**Source:** [github.com/BayramAnnakov/claude-reflect](https://github.com/BayramAnnakov/claude-reflect)

### pro-workflow

A comprehensive framework with persistent SQLite + FTS5 storage. "You correct Claude once -- it never makes the same mistake again."

- **Multi-point capture**: UserPromptSubmit hooks monitor corrections; PostToolUse failures suggest learnings; Stop handlers auto-capture `[LEARN]`-tagged blocks; `/learn-rule` for explicit extraction
- **Automatic reload**: SessionStart hooks load all stored learnings at session initiation
- **Contextual retrieval**: `/replay` surfaces past learnings relevant to current tasks via BM25 ranking
- **Session continuity**: `/handoff` generates structured session documents for seamless cross-session work
- **Scale**: 24 hook events, 29 scripts, 24 skills, 8 agents

Claims: "After 50 sessions, Claude barely needs correcting."

**Source:** [github.com/rohitg00/pro-workflow](https://github.com/rohitg00/pro-workflow)

### ChristopherA's Bootstrap Seed

A single ~1400-token prompt bootstraps a self-improving system with no pre-built infrastructure:

- After non-trivial work, triggers reflection: "What worked? What didn't? What pattern emerged?"
- Learnings captured in `.claude/learnings.md`; when referenced 2+ times, promoted to `.claude/rules/`
- Learnings capped at ~30 entries; overflow triggers consolidation (group into themes, promote patterns, archive integrated items)
- Rules capped at ~50 lines before splitting into "rule + process document"
- Uses `AskUserQuestion` at structural decisions so humans steer evolution

**Source:** [Self-Improving Claude Code Bootstrap Seed](https://gist.github.com/ChristopherA/fd2985551e765a86f4fbb24080263a2f)

### Three-Layer Memory Architecture

A more sophisticated persistent memory system with tiered storage:

- **Tier 1 (Auto-load)**: Compact CLAUDE.md (~150 lines), loads every session
- **Tier 2 (On-demand)**: `.memory/state.json` with unlimited capacity, queryable via MCP tools
- **Tier 3 (Hooks)**: Session-based capture at Stop, PreCompact, and SessionEnd events

Memory evolution through decay: architecture/pattern memories are permanent; progress memories decay over 7 days; context memories over 30 days. Below 0.3 confidence, memories remain searchable but excluded from CLAUDE.md. Every 10 extractions, Haiku consolidates overlapping entries and removes contradictions.

**Source:** [The Architecture of Persistent Memory for Claude Code](https://dev.to/suede/the-architecture-of-persistent-memory-for-claude-code-17d)

---

## 4. Hook-Based Learning and Enforcement

Hooks convert mistakes into automated prevention. The critical distinction: "Prompts are suggestions. Claude can be convinced to ignore them. Hooks are different." Hooks return exit code 2 to block operations unconditionally.

### Stop Hooks for Verification

Stop hooks run when the agent finishes responding and can enforce completeness:

- **Test gates**: Run `npm test`; block until tests pass
- **Build validation**: Execute `npm run build`; block on compilation errors
- **Lint enforcement**: Block with `eslint --max-warnings=0`
- **AI-based verification**: A prompt hook where a separate, fast model evaluates whether all requested tasks were completed

The `stop_hook_active` flag prevents infinite loops when Claude is already continuing from a prior block.

**Source:** [Claude Code Stop Hook: Force Task Completion](https://claudefa.st/blog/tools/hooks/stop-hook-task-enforcement), [Claude Code Hooks: Guardrails That Actually Work](https://paddo.dev/blog/claude-code-hooks-guardrails/)

### PostToolUse Hooks for Real-Time Correction

PostToolUse hooks fire after tool execution:
- Auto-format code after every Write/Edit (idempotent, safe to run always)
- Detect patterns in modified files (e.g., hardcoded secrets matching `(API_KEY|SECRET|TOKEN)\s*[=:]\s*["'][A-Za-z0-9_\-]{16,}`)
- Warn when test files change (catching the anti-pattern of rewriting tests to match broken implementation)

### The Learning Gap in Current Hooks

Current hooks are **reactive but not adaptive**. They enforce fixed rules but don't update those rules based on what they catch. A hook that blocks a secret leak today will block the same pattern tomorrow, but it won't learn to detect a *new* pattern of secret leakage. The learning still requires human observation.

---

## 5. The Continuous Coding Loop (Addy Osmani)

Addy Osmani documented the "Ralph Wiggum technique" -- a continuous loop architecture:

1. Pick next task from structured to-do
2. Agent implements
3. Validate with tests/checks
4. Commit if checks pass
5. Update task status and log learnings
6. Reset context and repeat

**Four channels of memory persist across iterations:**
- Git commit history (agents inspect diffs)
- Progress log (chronological attempts, pass/fail, discoveries)
- Task state (prd.json tracking remaining requirements)
- Semantic knowledge (AGENTS.md with accumulated patterns)

**Scaling pattern**: Planner agents assess and prioritize, Worker agents execute, Judge agents evaluate completion criteria. This prevents risk-averse behavior where individual agents make only tiny changes.

**Source:** [AddyOsmani.com - Self-Improving Coding Agents](https://addyosmani.com/blog/self-improving-agents/)

---

## 6. Research: Fully Automated Self-Improvement

### SICA (Self-Improving Coding Agent)

Published at ICLR 2025 SSI-FM workshop. The agent directly edits its own source code (prompts, heuristics, architecture), evaluates against benchmarks, and retains improvements.

- Iteratively benchmarks itself, computes utility score (success, cost, runtime)
- Uses best-performing historical version to propose modifications
- 17-53% performance gains on SWE-Bench Verified

Unlike ADAS (which separates meta-agent from target agent), SICA is both initiator and recipient of self-improvement.

**Source:** [A Self-Improving Coding Agent](https://arxiv.org/abs/2504.15228), [github.com/MaximeRobeyns/self_improving_coding_agent](https://github.com/MaximeRobeyns/self_improving_coding_agent)

### Yohei Nakajima's Taxonomy

Six mechanisms for agent self-improvement (synthesized from NeurIPS 2025 work):

1. **Self-reflection**: Critique own outputs, retry. Reflexion achieves ~91% pass rates on coding benchmarks.
2. **Self-generated curricula**: Create test-based tasks, solve them, use verified solutions as training data. Doubles tool-use performance.
3. **Learned self-correction**: Train on mistake-fix trace pairs. Small models approach larger baselines.
4. **Self-adapting models**: Generate natural-language edit instructions, convert to fine-tuning examples.
5. **Code self-edit**: Recursively improve own code (STO discovers beam search, simulated annealing without guidance).
6. **Embodied self-practice**: Combine supervised fine-tuning with self-practice RL.

The convergent insight: "Treat interaction traces as persistent, reusable structures (exemplars, skills, training data) that agents can query, refine, and compose."

**Source:** [Better Ways to Build Self-Improving AI Agents](https://yoheinakajima.com/better-ways-to-build-self-improving-ai-agents/)

### Telemetry-Driven Loops (Arize)

Harrison Chase (LangChain): "In software, the code documents the app; in AI, the traces do." The self-improvement loop requires:

1. Instrument code paths
2. Execute and collect telemetry
3. Agent queries traces to identify reasoning errors and tool-call loops
4. Run targeted evaluations against real trace data
5. Trace data guides refinement
6. Changes ship with supporting evidence

Without telemetry access, "the self-improvement loop breaks at its most critical point: verification."

**Source:** [Closing the Loop: Coding Agents, Telemetry, and the Path to Self-Improving Software](https://arize.com/blog/closing-the-loop-coding-agents-telemetry-and-the-path-to-self-improving-software/)

---

## 7. The Recursive Improvement Problem

### Configuration Drift

Self-modifying agent configurations face the same risks as any system that modifies its own state:

- **Accumulation**: Small changes compound until the original behavior is unrecognizable
- **Conflicting rules**: New rules may contradict existing ones without detection
- **Scope creep**: Rules proliferate until the configuration itself becomes a maintenance burden
- **Echo chamber**: Agent learns from its own outputs, reinforcing biases

91% of ML systems experience performance degradation without proactive intervention.

**Source:** [A Comprehensive Guide to Preventing AI Agent Drift Over Time](https://www.getmaxim.ai/articles/a-comprehensive-guide-to-preventing-ai-agent-drift-over-time/)

### Meta's HyperAgents Safety Architecture

Meta's research addresses self-modification safety through:

- **Alignment anchors**: Immutable core objectives that cannot be modified; all changes must serve these anchors
- **Self-representation layer**: Structured knowledge of own code as a semantic graph
- **Atomic deployment**: Version control, rollback capability, gradual rollouts
- **Formal verification**: Mathematical proofs of termination and resource bounds
- **Sandbox isolation**: No network access during self-modification

This remains experimental and non-production-ready.

**Source:** [HyperAgents: Self-Improving AI Systems from Meta Research](https://pooya.blog/blog/hyperagents-self-improving-ai-meta-research-2026/)

### Practical Guardrails for Self-Modifying Configurations

Based on the tools that exist today:

1. **Human review gate**: claude-reflect requires `/reflect` approval; ChristopherA's seed uses `AskUserQuestion`. No tool auto-commits rule changes without human review.
2. **Size constraints**: Cap learnings files, force consolidation, prevent unbounded growth
3. **Confidence decay**: Memory systems that deprecate stale learnings automatically
4. **Version control**: Rules in git, so `git diff` and `git revert` work
5. **Eval suites**: Test the rules themselves (Anthropic recommends starting with 20-50 tasks from real failures)

---

## 8. The SRE Postmortem Analogy

DevOps/SRE teams have solved this problem for operational systems: incident happens, postmortem runs, runbook updates, future incidents handled faster. The same pattern applied to coding agents:

- **Incident** = agent produces bad output or misses a requirement
- **Postmortem** = reflection prompt analyzing what went wrong
- **Runbook update** = CLAUDE.md or skill modification
- **Automated response** = hook that catches the pattern next time

AWS DevOps Agent demonstrates this in production: after resolving three DynamoDB throttling incidents, the Learning Agent identifies the recurring pattern and generates a "learned skill" that accelerates future investigations.

Teams are also "shifting left with all operational knowledge" -- taking runbooks and operational context and plugging them into coding agents at development time.

**Source:** [Leverage Agentic AI for Autonomous Incident Response with AWS DevOps Agent](https://aws.amazon.com/blogs/devops/leverage-agentic-ai-for-autonomous-incident-response-with-aws-devops-agent/), [Part 1: Death of the Toil: How AI Agents Are Replacing Traditional Runbooks](https://devops.com/part-1-death-of-the-toil-how-ai-agents-are-replacing-traditional-runbooks/)

---

## 9. The Specific Failure: Missing Explicit Requirements

The prompt for this research describes a concrete failure: 7 research agents missed a source the user explicitly asked about. A human caught the gap in seconds.

### Why Agents Miss Requirements

- **Context loss**: Long prompts get partially attended to; items late in the list or nested in subordinate clauses are missed
- **Satisficing**: Agents find "enough" sources and stop, rather than verifying coverage against the original request
- **No checklist verification**: Agents don't systematically cross-reference their output against input requirements

### Design Patterns That Catch This

**Requirement extraction hook**: Before execution, parse the user's prompt into an explicit checklist of requirements. Store as structured data.

**Completeness verification stop hook**: Before allowing the agent to stop, cross-reference output against the extracted checklist. Block if items are uncovered.

**Implementation sketch** (not production-tested):
```json
{
  "hooks": {
    "Stop": [{
      "type": "prompt",
      "prompt": "Review the user's original request and the agent's response. List each specific source, topic, or requirement mentioned in the request. For each, verify it appears in the response. If any are missing, return {\"decision\": \"block\", \"reason\": \"Missing: [list]\"}. If all covered, return {\"decision\": \"allow\"}."
    }]
  }
}
```

**Multi-agent verification**: A separate verification agent whose only job is coverage analysis. It receives the original prompt and the output, and returns a gap report. This mirrors the Judge agent pattern from Osmani's planner-worker-judge architecture.

**Eval-driven prevention**: Build a test case from the failure. "Given this prompt, does the output mention [specific source]?" Add it to the eval suite. Future changes to the research workflow must pass this regression test.

---

## 10. Kiro and Adjacent Tools

### Kiro's Steering Files

Kiro uses `.kiro/steering/` markdown files for persistent project knowledge. The `/knowledge` command provides cross-session persistent storage with semantic search. However, there is no documented mechanism for Kiro to automatically update its own steering files based on failures.

A feature request (Issue #6988) asks for "cross-session knowledge memory and topic-based session search," suggesting this is not yet built in.

**Source:** [Knowledge management - Kiro Docs](https://kiro.dev/docs/cli/experimental/knowledge-management/), [Feature Request #6988](https://github.com/kirodotdev/Kiro/issues/6988)

### Claude Code's Built-in Auto Memory

Claude Code's auto-memory system captures durable project knowledge without manual effort. It saves build commands, debugging insights, architecture notes, and style preferences. But this is *memory* (what Claude knows about the project), not *rules* (what Claude must do). The distinction matters: memory informs; rules constrain.

**Source:** [How Claude remembers your project](https://code.claude.com/docs/en/memory)

---

## 11. What's Missing: The Unsolved Problems

### No Closed-Loop Rule Testing

Nobody has a production system that:
1. Detects an agent failure
2. Proposes a new rule
3. Tests the rule against historical failures
4. Verifies it doesn't break existing behavior
5. Deploys it automatically

Steps 1-2 exist (claude-reflect, pro-workflow). Steps 3-5 are manual or absent. The eval infrastructure from Anthropic (20-50 test cases from real failures) is the closest thing to step 3, but it requires human curation.

### No Automated Rule Quality Assessment

Rules accumulate. Nobody has a system that measures whether a specific rule is still useful, whether it conflicts with other rules, or whether it's been superseded. The memory decay systems (confidence scores, time-based deprecation) are a proxy but don't verify actual effectiveness.

### No Cross-Project Learning

Each project's CLAUDE.md starts from scratch. There is no mechanism to identify patterns that apply across projects (e.g., "never mock the database" is a universal principle, not project-specific). Global `~/.claude/CLAUDE.md` is a manual approximation.

### The Verification Paradox

If an agent is bad enough to need self-correction, how do you trust it to write good corrections? The current answer is human review (claude-reflect's `/reflect` command, ChristopherA's `AskUserQuestion`). Full automation would require a verifier more capable than the agent being improved -- which either means a stronger model or formal verification methods.

---

## 12. Practical Recommendations

For teams wanting to move along the manual-to-automated spectrum:

**Level 1 (Today)**: Add Boris Cherny's discipline. When Claude does something wrong, say "Reflect on this mistake. Abstract and generalize the learning. Write it to CLAUDE.md." Review the addition. Commit it.

**Level 2 (Low effort)**: Add stop hooks that verify basic completeness. Run tests, check builds, lint. This catches a class of "agent thinks it's done but isn't" failures automatically.

**Level 3 (Moderate effort)**: Install claude-reflect or pro-workflow. Get semi-automated correction capture with human review gates. Build an eval suite from real failures (start with 20-50 cases).

**Level 4 (High effort)**: Build a completeness verification hook that cross-references output against input requirements. Implement the planner-worker-judge pattern for critical tasks. Add telemetry to trace agent behavior and feed it back.

**Level 5 (Experimental)**: Build a meta-agent that reviews accumulated rules for conflicts, redundancy, and effectiveness. Test rules against historical transcripts. This is where the field is headed but nobody has a mature implementation.

---

## Summary

The state of self-improving agent configurations in early 2026:

| Capability | Status |
|---|---|
| Human-triggered rule updates | **Established practice** (Boris Cherny pattern) |
| Agent-written rules from reflection prompts | **Working well** when human-reviewed |
| Semi-automated correction capture (hooks) | **Early tools exist** (claude-reflect, pro-workflow) |
| Stop hooks for completeness verification | **Production-ready** (built into Claude Code) |
| Persistent memory across sessions | **Built-in** (Claude Code auto-memory) |
| Fully automated self-improvement loops | **Research only** (SICA, HyperAgents) |
| Rule quality assessment and pruning | **Unsolved** |
| Cross-project learning transfer | **Manual only** (global CLAUDE.md) |
| Closed-loop rule testing | **Unsolved** |

The SRE postmortem analogy is the right mental model: every agent failure should produce a durable artifact (rule, hook, eval case, or skill update) that prevents recurrence. The manual version of this works today. Automating it fully remains an open problem, primarily because verifying the quality of self-generated rules is as hard as the original task.
