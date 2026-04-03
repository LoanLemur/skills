---
name: forge
description: >-
  Build, test, and iterate on agentic skills using a Ralph-style loop.
  Use when creating a new skill or enhancing an existing one. Takes a skill
  spec and acceptance criteria, then iterates with fresh context until all
  tests pass. DO NOT use for non-skill work.
argument-hint: <skill-name>
user-invocable: true
---

# Forge

Build or enhance a skill through automated iteration. Each cycle: draft →
test → evaluate → improve. Fresh context per test. Only surface to the
user when all tests pass or you're stuck.

## Input

$ARGUMENTS

This should be a skill name. Find the skill's entry in
`docs/research/28-skill-map.md` for its spec. If an existing version
exists in `~/.claude/skills/{name}/SKILL.md`, read it as the starting
point.

Also read:
- `docs/research/00-synthesis.md` — principles and pipeline
- `docs/research/08-agent-skills-standard.md` — the SKILL.md format spec
- `docs/research/24-skill-writing-craft.md` — how to write skills well
- The relevant craft analysis docs (docs/research/20-25) for patterns

## Phase 1: Understand the Skill

Before writing anything:

1. Read the skill map entry. Identify: job, input, output, key
   enhancements, failure modes.
2. If enhancing an existing skill, read the current version completely.
   Identify what works (keep), what's missing (add), and what contradicts
   the research (change).
3. Read the acceptance criteria (end of this file). These are the tests
   the skill must pass. Understand them before drafting.
4. Read the relevant research docs for this specific skill. The skill map
   and synthesis reference which docs matter.

State back what you understand: what this skill does, what its acceptance
criteria are, and your approach. Do not proceed without this.

## Phase 2: Draft the Skill

Write the SKILL.md following these craft principles (from docs 20-25):

**Format:**
- YAML frontmatter: `name`, `description` (triggering conditions ONLY —
  never summarize the workflow in the description), `user-invocable`,
  `argument-hint`
- Markdown body under 500 lines
- Heavy reference material goes in `references/`, not inline
- Templates and boilerplate go in `assets/`

**Description field (make-or-break):**
- 50-150 words
- Only triggering conditions: "Use when...", "DO NOT TRIGGER when..."
- Use task verbs ("writing", "reviewing"), not technology nouns
- If the description summarizes the workflow, the agent will follow the
  summary and skip the body

**Prose:**
- Imperative, no hedging
- Explain WHY, not just WHAT — Claude generalizes better from reasoning
  than from commands
- Show, don't tell — Incorrect/Correct pairs where applicable
- No inflated language (per CLAUDE.md — no "critical", "crucial", etc.)
- Every instruction must pass the test: "Would removing this cause the
  agent to make a mistake?" If not, remove it.

**Structure:**
- One-sentence opening that states what the skill does
- Phases with clear inputs/outputs/gates
- Output format with explicit constraints (templates, section limits)
- Explicit "stop and report" failure mode
- Follow CLAUDE.md output standards and posture throughout

**Anti-patterns to avoid:**
- Persona framing ("You are a...") — no evidence this helps, slight
  evidence it hurts (PRISM paper, -0.65 on coding tasks)
- Process summaries in the description field
- Advisory instructions for things that can be enforced structurally
- Duplicating what CLAUDE.md already covers
- Generic instructions ("be thorough", "do your best")

Write the skill. Include all supporting files (references/, assets/).

## Phase 3: Test

Run three tests. Each test MUST use a sub-agent with fresh context —
the agent that wrote the skill cannot evaluate it.

### Test A: Adversarial

Spawn a sub-agent. Provide it with:
- The skill's SKILL.md (full content)
- The instruction: "You are testing this skill for loopholes. Read the
  instructions and find ways to: skip steps, produce shallow output,
  declare done prematurely, avoid the hard parts, generate verbose
  filler, or violate the spirit of the instructions while following the
  letter. Report every loophole you find. If the skill is tight, say so."

Record findings.

### Test B: Two-Claude Evaluation

Spawn a sub-agent. Provide it with:
- The skill's SKILL.md
- The skill map entry (the goal-level spec)
- The acceptance criteria from this file
- The instruction: "Evaluate this skill against the acceptance criteria.
  For each criterion, state PASS or FAIL with specific evidence. Do not
  give the benefit of the doubt — if a criterion is ambiguously met,
  it's a FAIL."

Record findings.

### Test C: Comparison (for enhanced skills only)

If an existing version of the skill exists, spawn a sub-agent. Provide:
- The old SKILL.md
- The new SKILL.md
- The instruction: "Compare these two versions. For each difference,
  state which version is better and why. Identify anything the old
  version did well that the new version lost. Identify improvements in
  the new version. Be specific — cite line numbers and content."

Record findings.

## Phase 4: Evaluate

Read all test results. Categorize:
- **PASS** — All three tests report no issues
- **FAIL with clear fixes** — Tests identified specific problems with
  obvious solutions. Proceed to Phase 5.
- **FAIL with unclear fixes** — Tests identified problems but the
  solution isn't obvious. Surface to the user with findings and ask
  for direction.
- **STUCK** — Multiple iteration cycles haven't resolved the issues.
  Surface to the user with the full history.

## Phase 5: Improve

Fix the issues identified in Phase 4. For each fix:
- State what the test found
- State what you're changing
- State why this fixes it

Then return to Phase 3 (test again). This is the Ralph loop — iterate
until PASS or STUCK.

Never weaken a test to make the skill pass. If a test seems wrong,
surface to the user.

## Phase 6: Deliver

When all tests pass:

1. Present the final skill to the user:
   - The SKILL.md (full content)
   - Any supporting files (references/, assets/)
   - Test results summary (what was tested, what passed)
   - Iteration history (how many cycles, what was fixed)
2. Ask the user to review and approve.
3. On approval, write the files to the target location.

## Acceptance Criteria (for skills built by /forge)

Every skill must:

1. **Follow the Agent Skills standard.** SKILL.md with YAML frontmatter
   (`name`, `description`). Directory structure with optional references/,
   scripts/, assets/. Name matches directory name.

2. **Description triggers correctly.** Contains only triggering conditions.
   Does not summarize the workflow. 50-150 words.

3. **SKILL.md under 500 lines.** Heavy reference material in references/.

4. **No sycophantic language in any instruction or output template.**
   No praise, compliments, inflated language, or hedging.

5. **Explicit output format with constraints.** Templates, section limits,
   or structured formats that prevent verbose filler.

6. **Explicit "stop and report" failure mode.** The skill defines when
   and how to halt and surface problems to the user.

7. **Passes adversarial testing.** An adversarial agent cannot find
   loopholes to skip steps, produce shallow output, or declare done
   prematurely.

8. **Passes two-Claude evaluation.** A separate evaluator confirms the
   skill meets its goal-level spec from the skill map, not just its
   task-level instructions.

9. **Matches or exceeds existing version** (for enhanced skills).
   Comparison test confirms no regressions and identifies clear
   improvements.

## Rules

- Never weaken tests to make a skill pass.
- Never declare a skill done without running all three tests.
- Surface to the user when stuck, not when "almost done."
- The iteration loop is the point — more cycles is expected, not failure.
- Fresh context for every test. The agent that wrote the skill cannot
  evaluate it.
