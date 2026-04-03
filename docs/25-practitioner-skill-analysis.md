# Practitioner Skill Analysis: /define, /design, /execute (and supporting cast)

Analysis of seven hand-crafted skills from an experienced Rails developer, compared against patterns from Anthropic's official skills, Vercel's plugin/agent-skills, and Trail of Bits' security harness.

---

## Part 1: Individual Skill Analysis

### /define (242 lines)

**Structure.** Six phases plus a rules footer. Phases progress linearly: understand problem, understand codebase, define solution, edge cases, behavior inventory, three-pass review, create GitHub issue. The structure mirrors Anthropic's mcp-builder (Phase 1-4 linear workflow) but adds an explicit adversarial review phase that no Anthropic skill includes.

**Prose style.** Imperative, specific, opinionated. "State findings as recommendations, not questions" -- this matches both Anthropic's and Vercel's preferred voice. The skill avoids hedging. Where it explains reasoning ("Implementation details belong in the design phase, not here"), the explanation is one sentence, not a paragraph. Tighter than Anthropic's skill-creator, which spends paragraphs on philosophy.

**Progressive disclosure.** Everything is inline. No references/, no scripts/, no assets/. At 242 lines this is within Anthropic's recommended 500-line ceiling, so this is acceptable. But the issue template (Phase 6) could be a file in assets/ -- it is structural boilerplate that consumes ~40 lines of instruction context on every invocation.

**Role assignment.** "You are a product manager and project manager." This is a pattern absent from Anthropic's skills, which never assign personas. Vercel's skills also avoid it. The research is mixed on role prompting effectiveness with current models -- it was more impactful with earlier generations. However, the role here is functional, not decorative: it scopes what the agent should and should not do (no code, no architecture). That scoping function is valuable even if the persona framing is not.

**Multi-agent integration.** Three-pass review: DHH, Devil, DHH. The DHH and Devil subagents are custom agents defined in `.claude/agents/`. This is the most distinctive pattern in these skills and has no direct analog in the public ecosystem. Anthropic's skill-creator uses subagents for eval running, but not for adversarial review of its own output. The three-pass pattern (expert, adversary, fresh expert) implements the synthesis spec's P5 (selective context transfer) more rigorously than any public skill.

**Artifact management.** Output is a GitHub issue, not code. This is unusual -- most public skills produce code or configuration. The issue becomes the contract between phases. This is a strong pattern: the artifact is reviewable, persistent, and decoupled from implementation.

**Rigidity vs flexibility.** Phase 1 is flexible (ask questions, challenge the premise). Phase 6 is rigid (exact issue template). The behavior inventory (Phase 5) is explicitly called "the contract between define and design." This graduated rigidity is well-calibrated: loose where exploration matters, tight where handoff matters.

**Exceptional.** The "Challenge the premise" section (lines 36-69) is the best requirements-gathering prompt I have seen in any skill. It does not ask generic questions -- it provides eight specific areas to probe, then instructs the agent to state findings as recommendations rather than questions. This forces the agent to form an opinion before surfacing it, which produces dramatically better output than "have you considered...?" The instruction "Then ask 2-3 pointed questions -- not generic ones, but the specific uncomfortable question this particular feature raises" is precise enough to be actionable and vague enough to apply to any feature.

**Could improve.**
1. The issue template (Phase 6) should be an asset file. It is reference material, not instruction.
2. The `argument-hint` field uses a bracket syntax `[feature or change description]` that differs from /design and /execute's angle-bracket syntax. Minor inconsistency.
3. No mention of proportional ceremony. A one-sentence bug fix should not go through six phases. The synthesis spec (P2) explicitly addresses this: "Trivial: No spec. Just do it." The skill could add a gate at the top: "If the change is a bug fix or config tweak with obvious scope, skip to Phase 6 and create the issue directly."

---

### /design (263 lines)

**Structure.** Six phases. Input parsing, understand spec, explore codebase (with three-pass review), propose implementation, three-pass adversarial review, finalize with user, update issue. Two separate three-pass review cycles -- one for architectural direction, one for the proposed design. Six total subagent invocations per skill execution.

**Prose style.** Same imperative voice as /define. Notably concise in the data model section: "For each table, justify why it exists. Always ask: can this be a column on an existing model instead of a new table?" This is the CLAUDE.md "deep modules" and "simplicity first" philosophy compressed into two sentences.

**Progressive disclosure.** Again fully inline, no external files. At 263 lines this is well within budget.

**Role assignment.** "You are a software architect." Same functional scoping as /define -- tells the agent what it is responsible for and, by omission, what it is not.

**Multi-agent integration.** Six subagent reviews (two three-pass cycles). This is expensive. Each DHH/Devil invocation is a separate context window. For a medium feature, this skill spawns 6 subagents before any code is written. The synthesis spec recommends review but does not prescribe this volume. The question is whether the second three-pass cycle (Phase 4) catches enough issues beyond the first (Phase 2) to justify the cost. In practice, Phase 2 reviews the architectural direction (what approach?) while Phase 4 reviews the detailed design (is this approach correct?). These are genuinely different questions, so both cycles likely earn their keep.

**Capability ordering.** The capabilities section (lines 130-148) is where the CLAUDE.md philosophy ("decompose by behavior, not by architectural layer") is operationalized. The instruction "Each capability is the smallest independently valuable unit" directly implements synthesis P6. The explicit caveat "The executing agent determines commit boundaries per CLAUDE.md conventions -- this list is a starting point, not a contract" is mature: it prevents the plan from becoming a straitjacket.

**Testing strategy.** Lines 159-166 are remarkable: "Default stance: most code doesn't need tests." This is the CLAUDE.md testing philosophy distilled. It directly contradicts the synthesis spec's P3 ("Never skip tests entirely") but aligns with the CLAUDE.md's "Tests that don't provide confidence are pure liability." The practitioner is more aggressive about test minimalism than the synthesis recommends. Both positions have evidence behind them. The CLAUDE.md position optimizes for a codebase with strong conventions and an experienced developer reviewing output. The synthesis position optimizes for teams where agents might silently introduce bugs. Both are defensible given their contexts.

**Exceptional.** The "When there's a clear best approach" vs "When there are genuinely different options" branching (lines 89-105) is excellent. Most planning skills always present multiple options, which wastes time when the answer is obvious. This skill adapts its ceremony to the situation -- present one approach when one is clearly right, present options only when they genuinely compete.

**Could improve.**
1. The "propose concrete names" instruction (line 150) is a high-value low-cost addition that should be in bold or its own section. Naming reveals design problems early, and burying this instruction in a paragraph reduces its salience.
2. No explicit connection to external API documentation. If the feature touches Stripe/Plaid/etc., the skill does not instruct the agent to fetch current API docs. The mcp-builder skill uses WebFetch for this.

---

### /execute (214 lines)

**Structure.** Five phases. Understand plan, setup (branch), execute commit-by-commit (with three-pass review per commit), final verification (with three-pass final-state review), report. The commit-by-commit loop (Phase 3) is the core -- it is a repeating cycle of implement, verify, review, commit, check alignment.

**Prose style.** The most directive of the three. Longer paragraphs than /define or /design, but the added length earns its place -- implementation instructions need more specificity than planning instructions. The "Study existing patterns before writing new code" paragraph (lines 66-67) is a concrete operationalization of "the codebase is the style guide."

**Progressive disclosure.** Inline. No external files. At 214 lines, the shortest of the three core skills.

**Multi-agent integration.** Three-pass review per commit, plus a three-pass final-state review. For a feature with 5 commits, this is 18 subagent invocations. This is the most review-intensive implementation skill in any public or private corpus I have analyzed. The per-commit review prompt (lines 89-107) is exceptionally well-crafted: it provides structural prompts ("Before reviewing code quality, answer two structural questions..."), explicit framing for cumulative diffs, and specific things to look for. This is selective context transfer (synthesis P5) done right.

**Artifact management.** Commits are the artifacts, not documents. Deviations from the plan are recorded as GitHub issue comments, not edits to the issue body. This "comments over edits" rule (line 212) preserves the original design as a reference while documenting what actually happened. This is a pattern I have not seen elsewhere and it is smart -- it provides an audit trail without corrupting the source of truth.

**Testing decisions.** Lines 73-74: "You own testing decisions. The issue's testing strategy identifies where the design anticipated non-trivial logic. Use it as a signal, not a directive." This gives the executing agent authority to override the design's testing recommendations based on actual code. This is mature -- it acknowledges that the design phase cannot perfectly predict which code will be trivial.

**Git workflow.** The "Handling Mistakes" section (lines 197-204) is specific and correct: fixup+autosquash for clean history. The warning about not rebasing pushed commits shows production experience.

**Exceptional.** The structural prompt injected into every review (lines 100-107) is the best review framing I have seen. It forces the reviewer to answer structural questions before code quality questions, which prevents the common failure mode of reviewing code quality in a commit that should not exist in its current form.

**Could improve.**
1. No explicit context management. A 5-commit feature with 18 subagent invocations and extensive code reading will consume significant context. There is no instruction to compact or clear between commits. The synthesis spec (P7) recommends compacting every 25-30 minutes during intensive work.
2. The design completeness check (Phase 1, step 4) duplicates work that /design already did. If the design skill produced a verified behavior inventory, re-verifying it here is redundant ceremony. However, given that time may pass between /design and /execute (and the issue might be edited), re-verification is defensible.
3. No mention of TDD or test-first. The skill says "write the code for this commit" and includes tests as part of the atomic unit, but does not specify whether tests come first or last. The synthesis spec (P3) recommends test-first for known requirements. Given the CLAUDE.md's aggressive test minimalism, this omission may be intentional.

---

### /pr-review (340 lines)

The longest skill. A complete workflow for analyzing PR review comments, forming independent opinions, drafting replies, and optionally posting fixup diffs inline. The "Resolve only" pattern for trivial fixes (no text reply, just fix and resolve) is a practical optimization that reduces PR noise. The verification pass after posting (lines 294-313) -- spawning a review agent to check its own posted replies for relevance and accuracy -- is a self-correction loop that no public skill implements.

### /review (500 lines)

The most sophisticated skill in the set. An interactive commit-by-commit code review with history surgery (rebase, split, squash, reorder, drop). The "annotated diff walkthrough" format (lines 209-256) is a presentation pattern: show diff chunks interleaved with commentary, not diff-then-commentary. The anti-narration bias section (lines 260-280) and convention evaluation checklist (lines 282-315) are battle-tested guards against the most common LLM review failure mode: accurately describing code without judging it.

The conflict handling section (lines 388-420) is production-grade: record HEAD before rebase, use range-diff afterward to identify non-trivial changes, verify commit messages still match content after surgery. This level of git operational detail is absent from every public skill.

### /qa (333 lines)

Browser-based QA testing using MCP tools (chrome extension). The testing heuristics section (lines 239-315) is a comprehensive edge-case checklist organized by category (forms, navigation, authorization, data display, Stripe). The Stripe test data section (lines 275-311) with specific test card numbers and bank account flows shows domain expertise. The JavaScript dialog handling section (lines 66-90) addresses a real browser automation gotcha.

### /retro (183 lines)

Engineering retrospective from git history. Synthesizes commits into themed narratives for stakeholders. The Slack posting integration with Block Kit formatting is a nice touch. The "flag if fix ratio exceeds 50%" heuristic is a useful signal.

---

## Part 2: Cross-Cutting Analysis

### Pipeline Implementation: SPECIFY --> PLAN --> IMPLEMENT --> REVIEW

The three core skills map directly to the synthesis pipeline:

| Pipeline Phase | Skill | Output |
|---|---|---|
| SPECIFY | /define | GitHub issue (problem, behaviors, decisions, constraints) |
| PLAN | /design | Implementation plan appended to same issue (data model, capabilities, notes) |
| IMPLEMENT | /execute | Commits on a feature branch, deviations as issue comments |
| REVIEW | Built into /execute (per-commit + final-state) and /review (interactive) |

The pipeline is well-integrated. Each skill's output is the next skill's input, and the GitHub issue is the persistent artifact that carries context across phases. The behavior inventory created in /define is the contract verified in /design and cross-referenced in /execute. This traceability from requirement to commit is stronger than any public skill pipeline.

**What is missing from the pipeline:**
- **Verification/completion skill.** The synthesis spec recommends a dedicated verification skill (Skill 6) that cross-references output against original requirements. /execute has a final verification phase, but it is embedded in the implementation skill rather than being an independent gate. A separate /verify that reads the issue's behavior inventory and checks the final diff against it would catch drift that the implementing agent might be anchored to.
- **Spike/exploration path.** The synthesis spec's P2 identifies that unknown problems benefit from building first, then specifying. These skills assume the standard pipeline. There is no /spike skill for when "writing a spec feels like guessing." Adding one -- or adding a gate at the top of /define that detects exploration-mode signals and recommends spiking first -- would address this gap.
- **Proportional ceremony.** A typo fix should not go through /define --> /design --> /execute. The skills do not address this. Adding a triage step ("Is this trivial? Skip to implementation.") would prevent over-specification of small tasks.

### Red Team Concerns

**Exploration mode.** Not addressed. These skills are designed for known problems with clear requirements. The synthesis spec's exploration pipeline (SPIKE --> LEARN --> SPECIFY --> IMPLEMENT --> REVIEW) has no implementation here.

**TDD ceremony vs signals.** Addressed well. The skills explicitly avoid prescribing TDD ceremony. /execute says "You own testing decisions" and /design says "Default stance: most code doesn't need tests." Tests are treated as signals, not ceremony. This aligns with the CLAUDE.md's aggressive position and the red team's finding that strict TDD instructions can increase regressions when applied naively to agents.

**Context management.** Not addressed. No skill mentions compacting, clearing, or managing context window utilization. For /execute in particular, which can run for extended periods across many commits, this is a gap.

### Self-Improvement Loop

The /retro skill analyzes what shipped but does not feed learnings back into CLAUDE.md or skill files. It is a reporting tool, not a self-improvement mechanism. The synthesis spec's Skill 9 (Self-Improvement/Retrospective) recommends "Capture learnings, update rules/skills, verify improvement." These skills are at maturity level 1 (manual) -- the human spots issues and updates instructions by hand.

The three-pass review pattern is a form of within-session self-improvement: the Devil challenges, the second DHH incorporates learnings. But this does not persist across sessions.

### Portability

These skills are moderately portable:

- **Claude Code specific:** Subagent spawning via `subagent_type: dhh` and `subagent_type: devil` is a Claude Code extension. The custom agents in `.claude/agents/` are Claude Code features. No other tool supports this.
- **GitHub CLI dependency:** Every skill uses `gh` for issue/PR operations. This works in any CI environment but not in editors that lack terminal access.
- **Rails specific:** The CLAUDE.md conventions, the DHH agent's checklist, and the /review evaluation dimensions are tuned for Rails. The skill structure itself is framework-agnostic, but the review criteria are not.
- **Agent Skills standard compliant:** The SKILL.md format with YAML frontmatter follows the standard. The skills would install via `npx skills add` if published. The `argument-hint` frontmatter field is a Claude Code extension.

### Craft Quality Comparison

**vs Anthropic's skills.** These practitioner skills are more opinionated, more structured, and more integrated with each other. Anthropic's skills are standalone -- each is self-contained and does not assume other skills exist. These practitioner skills form a pipeline where each skill's output feeds the next. The three-pass adversarial review pattern is more rigorous than anything in Anthropic's corpus. Anthropic's skills are better at progressive disclosure (reference files, scripts as black boxes). These practitioner skills put everything inline.

**vs Vercel's skills.** Vercel's skills are rule libraries and procedural dispatchers -- they do not orchestrate multi-phase workflows. The practitioner skills are workflow orchestrators that happen to contain rules. These are fundamentally different skill types. Vercel's skills are better at triggering (explicit positive and negative triggers in descriptions) and at progressive disclosure (rules/ directories, compiled AGENTS.md). The practitioner skills are better at multi-agent review and pipeline integration.

**vs Trail of Bits.** Trail of Bits focuses on prevention (deny lists, hooks) rather than orchestration. The practitioner skills focus on process quality. These are complementary, not competing. The practitioner could benefit from Trail of Bits' deterministic enforcement patterns -- for example, a hook that blocks `git commit` without a passing test suite, rather than relying on the agent to remember to run tests.

### CLAUDE.md Alignment

The skills enforce the CLAUDE.md philosophy well:

- **Fat models, skinny controllers:** The DHH agent's convention checklist explicitly checks this.
- **Strategic over tactical:** The /define skill's "Challenge the premise" section forces strategic thinking before tactical specification.
- **Capability-ordered decomposition:** /design produces capability-ordered lists, /execute implements them in order.
- **Testing philosophy:** Both /design and /execute default to minimal testing, matching the CLAUDE.md's "Tests that don't provide confidence are pure liability."
- **Commit practices:** /execute enforces atomic, complete, coherent commits with the correct conventional commit format.
- **No architecture astronautics:** The Devil agent challenges unnecessary abstractions. The DHH agent checks for Rails idiom conformance.

One tension: the CLAUDE.md says "No dependency injection, repository pattern, or hexagonal/clean architecture" and "Rails IS the architecture." The /design skill's "You are a software architect" role could encourage over-architecture. In practice, the DHH reviews likely counterbalance this, but the role assignment works slightly against the CLAUDE.md philosophy.

---

## Part 3: Recommendations

### High-Value, Low-Effort

1. **Extract issue templates to assets/.** The GitHub issue template in /define (Phase 6) and the implementation plan template in /design (Phase 6) are structural boilerplate. Moving them to `assets/issue-template.md` and `assets/plan-template.md` frees ~60 lines of instruction context across the two skills.

2. **Add proportional ceremony gates.** A 3-line gate at the top of /define: "If this is a bug fix, typo, or config change with obvious scope, skip directly to Phase 6." Similarly for /design: "If the issue describes a single-commit change with no data model changes, skip to Phase 5."

3. **Add context management instructions to /execute.** After every 3-4 commits (or when the agent has been running for 20+ minutes), instruct it to compact. The synthesis spec recommends this and the omission is the single most likely failure mode in long-running /execute sessions.

4. **Standardize argument-hint syntax.** /define uses `[brackets]`, /design and /execute use `<angle-brackets>`. Pick one.

### Medium-Value, Medium-Effort

5. **Create a /spike skill.** A lightweight skill for exploration-mode work: build a throwaway prototype, capture what you learned, produce a rough spec for /define to formalize. This addresses the synthesis spec's exploration pipeline gap.

6. **Add negative triggers to descriptions.** Following Anthropic's claude-api pattern: "DO NOT TRIGGER when: the change is a one-line fix, the requirements are already in the issue, etc." This prevents over-triggering.

7. **Add a post-execution verification step.** Either as a standalone /verify skill or as an enhanced Phase 4 in /execute: spawn a fresh-context agent that reads only the issue's behavior inventory and the final `git diff main...HEAD`, then reports which behaviors are covered and which are not. This implements synthesis Skill 6.

### Lower Priority

8. **Consider reducing review volume in /design.** Six subagent invocations before any code is written is expensive. The Phase 2 three-pass review (architectural direction) could be reduced to a single DHH review when the approach is obviously the only option. The branching already exists ("When there's a clear best approach") -- extend it to the review cycle.

9. **Externalize the DHH and Devil agent definitions.** These agents are currently in `.claude/agents/` which is correct for Claude Code. If the skills are ever published, the agents would need to be bundled with the skills (in an `agents/` directory per skill, as Anthropic's skill-creator does).

10. **Add a self-improvement hook to /retro.** After presenting the retrospective, ask: "Did any patterns in this period's work suggest a CLAUDE.md rule update or skill improvement?" This moves from maturity level 1 (manual) toward level 2 (human-triggered).

---

## Part 4: Summary Assessment

These are among the best-crafted agentic development skills I have analyzed. The pipeline integration (issue as persistent artifact, behavior inventory as contract, comments-over-edits for deviations) is more sophisticated than anything in the public ecosystem. The three-pass adversarial review pattern with custom DHH/Devil agents is a genuine innovation that implements the research literature's recommendations for selective context transfer and adversarial review more rigorously than any published skill.

The primary gaps are: no exploration-mode pathway, no explicit context management, no proportional ceremony scaling, and no self-improvement loop. These are addressable without restructuring the existing skills.

The craft quality is high. The prose is tight, opinionated, and specific. Instructions encode judgment, not just procedure. The skills know what they are not responsible for ("Never write code" in /define and /design; "Comments over edits" in /execute). The CLAUDE.md philosophy is consistently enforced through both the skill instructions and the review agents.

If Anthropic's skills represent "how to write skills well" and Vercel's represent "how to scale skills across teams," these practitioner skills represent "how to build a complete development workflow from skills." They are the most complete implementation of the SPECIFY --> PLAN --> IMPLEMENT --> REVIEW pipeline in any skill corpus currently available.
