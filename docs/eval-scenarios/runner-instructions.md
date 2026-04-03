# Behavioral Eval Runner Instructions

How to run the full evaluation suite for the four agentic development
skills: /define, /design, /execute, /review.

## Prerequisites

- A working copy of the Splitwise Lite test repo (Express/Node/SQLite)
- The skills installed and available to Claude Code agents
- `gh` CLI authenticated for GitHub issue creation
- Ability to spawn Claude Code agents (sub-agents or separate sessions)

## General Pattern

Each skill evaluation follows the same structure:

1. **Skill agent** — Runs the skill with scenario input, produces output
2. **Evaluator agent** — Grades the output against the rubric

The skill agent and evaluator agent must be separate sessions with
independent context. The evaluator must not have seen the skill agent's
reasoning process — only its output.

## Running Each Scenario

### 1. /define

**Setup:** No repo setup needed. The skill agent needs read access to
the Splitwise Lite codebase.

**Skill agent instructions:**
- Start in the Splitwise Lite repo directory
- Provide the user prompt from `scenario-define.md`
- The agent should run /define with the prompt
- Since /define is interactive (asks clarifying questions), simulate
  user responses that are cooperative but do not volunteer information
  the agent should discover on its own. Approve reasonable
  classifications and directions. Answer clarifying questions honestly
  based on the scenario context.
- Capture the final GitHub issue body (the output of /define)

**Evaluator agent instructions:**
- Provide the captured issue body
- Provide the grading rubric from `scenario-define.md`
- Instruct: "Score each rubric item as PASS / PARTIAL / FAIL with a
  one-sentence explanation. Do not infer intent — score only what is
  explicitly present in the output."
- Capture the score sheet

### 2. /design

**Setup:** Create a GitHub issue in the test repo with the spec from
`scenario-design.md` (the "Input: Completed /define Spec" section).

**Skill agent instructions:**
- Start in the Splitwise Lite repo directory
- Provide the issue number/URL as input to /design
- Simulate user responses: confirm the behavior inventory when
  presented, approve the approach when recommended, approve the final
  plan
- Capture the implementation plan appended to the issue

**Evaluator agent instructions:**
- Provide the implementation plan
- Provide the original spec (so the evaluator can check traceability)
- Provide the grading rubric from `scenario-design.md`
- Instruct: "Score each rubric item as PASS / PARTIAL / FAIL with a
  one-sentence explanation."
- Capture the score sheet

### 3. /execute

**Setup:**
- Create a GitHub issue with both the spec (from `scenario-design.md`)
  and the implementation plan (from `scenario-execute.md`)
- Ensure the test repo is on `main` with a clean working tree
- Ensure tests pass: `npm test`

**Skill agent instructions:**
- Start in the Splitwise Lite repo directory
- Provide the issue number/URL as input to /execute
- Simulate user responses: confirm the capability summary, approve the
  branch name
- Let the agent work through all commits
- Do not intervene unless the agent asks a question
- Capture the final branch state

**Evaluator agent instructions:**
- Start in the Splitwise Lite repo directory on the feature branch
- Run `git log main..HEAD --oneline` to see the commits
- Run `npm test` to verify tests pass
- For each commit, run `git show <sha>` to inspect the diff
- Provide the grading rubric from `scenario-execute.md`
- Provide the implementation plan for reference
- Instruct: "Score each rubric item as PASS / PARTIAL / FAIL with a
  one-sentence explanation. Run commands to verify — do not take claims
  at face value."
- Capture the score sheet

### 4. /review

**Setup:** This scenario requires a branch with intentional problems.
Build the branch as described in `scenario-review.md`:

1. Start from `main` in the Splitwise Lite repo
2. Create branch `feature/balance-calculation-review`
3. Create the 5 commits described in the scenario, including all
   intentional problems
4. Ensure the code compiles and basic tests pass (the problems are
   subtle, not syntax errors)

Building this branch is the most involved setup step. A setup agent
can implement it by following the commit descriptions literally. The
problems must be present but not cartoonishly obvious — they should
require careful reading to catch.

**Skill agent instructions:**
- Start in the Splitwise Lite repo on the `feature/balance-calculation-review` branch
- Create a GitHub issue with the spec from `scenario-design.md` for
  the /review agent to reference
- Run /review with the issue number
- Simulate the reviewer: say "structure looks good, let's go commit by
  commit" after the overview (to test whether the overview catches
  structural issues proactively), then "next" after each commit review
- Capture the full review output

**Evaluator agent instructions:**
- Provide the full review output
- Provide the planted problems list from `scenario-review.md`
- Provide the grading rubric from `scenario-review.md`
- Instruct: "Score each rubric item as PASS / PARTIAL / FAIL with a
  one-sentence explanation. The planted problems are the ground truth —
  a good review catches all of them."
- Capture the score sheet

## Collecting Results

After all four evaluations, produce a summary table:

```
Skill     | Category              | Score   | Notes
----------|-----------------------|---------|------
/define   | Problem Statement     | X/3     |
/define   | Behavior Inventory    | X/6     |
/define   | Edge Cases            | X/6     |
/define   | Constraints           | X/3     |
/define   | Key Decisions         | X/3     |
/define   | Spec Completeness     | X/3     |
/design   | Data Model            | X/5     |
/design   | Capabilities          | X/5     |
/design   | Traceability          | X/2     |
/design   | Technical Decisions   | X/3     |
/design   | Implementation Notes  | X/3     |
/design   | Plan Completeness     | X/3     |
/execute  | Commit Correspondence | X/2     |
/execute  | Commit Atomicity      | X/3     |
/execute  | Commit Messages       | X/4     |
/execute  | Test Quality          | X/5     |
/execute  | Correctness           | X/6     |
/execute  | Follows Patterns      | X/3     |
/review   | Mixed Commit          | X/2     |
/review   | Bad Message           | X/2     |
/review   | Missing Validation    | X/2     |
/review   | Missing Test          | X/1     |
/review   | Spec Divergence       | X/2     |
/review   | Division-by-Zero      | X/2     |
/review   | Review Quality        | X/4     |
```

Scoring: PASS = 1, PARTIAL = 0.5, FAIL = 0. Express each category as
points earned / points possible.

## Notes

- Each skill agent session should start clean. Do not reuse a session
  that ran a previous skill.
- The evaluator agent must not share context with the skill agent. This
  is the entire point — independent evaluation.
- If a skill agent fails catastrophically (crashes, refuses to run,
  produces nothing), score all rubric items as FAIL and note the failure
  mode.
- The /review scenario is the hardest to set up because it requires
  building a branch with specific subtle problems. Invest time in
  getting the setup right — if the problems are too obvious or too
  hidden, the evaluation loses signal.
