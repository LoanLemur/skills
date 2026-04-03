---
name: execute
description: >-
  Execute an implementation plan from a GitHub issue. Use when an issue
  has an approved implementation plan with capabilities ready to build.
  Use after /design or on any issue with a clear capability list and
  data model. Use when the user says "build this", "implement this", or
  "execute the plan". DO NOT use when the issue lacks an implementation
  plan, when requirements are still being defined, or when the user
  wants to design before building.
argument-hint: <github-issue-url-or-number>
user-invocable: true
---

# Execute

Implement a feature end-to-end from a GitHub issue. Work commit by
commit, review each commit independently, and maintain clean git history.

## Input

$ARGUMENTS

This should be a GitHub issue URL or number. If not provided, ask the
user which issue to implement.

## Phase 1: Understand the Plan

1. Fetch the issue using `gh issue view`. Read the entire body.
2. Identify:
   - The **design** — "How it works" narrative, key decisions, edge cases
   - The **data model** — tables, columns, types, constraints
   - The **capabilities** — the ordered list to implement
   - The **implementation notes** — patterns, state transitions, details
   - The **testing strategy** — where the design anticipated non-trivial logic
   - Any **outstanding questions** needing resolution
3. If there are outstanding questions or ambiguities, ask the user
   before proceeding. Do not guess at unresolved decisions.
4. **Design completeness check.** Cross-reference the design ("How it
   works") against the capability list. Extract every distinct behavior
   — each user interaction, state transition, or external integration.
   Verify each is covered by a capability. If the issue includes a
   behavior inventory, use it as the checklist. If not, construct one
   from the design narrative.

   If any behavior has no corresponding capability, stop. Ask the user
   whether to add it, defer it, or confirm it's already covered.
5. Present a summary to the user:
   - The behavior inventory (numbered)
   - The capability ordering (with behavior mapping)
   - Any gaps found
   - Wait for confirmation before proceeding.

## Phase 2: Setup

1. Create a feature branch off `main`:
   ```
   git checkout main && git pull && git checkout -b {branch-name}
   ```
   Use the branch name the user specifies, or ask.
2. Confirm the branch is clean and ready.

## Phase 3: Execute — Commit by Commit

Work through the plan one commit at a time.

### 3a. Implement

Write the code for this commit. Three sources of guidance:
1. **The capability description** — what this delivers and constraints
2. **The implementation notes** — patterns, conventions to follow
3. **The design section** — "How it works" is the source of truth for
   behavior; refer back for any questions about what the code should do

**Study existing patterns before writing new code.** Before creating any
new file, read 2-3 existing files of the same type and extract the
pattern. Before writing any new spec, read 2-3 existing specs of the
same type. Before adding query logic to a controller, read the model's
existing scopes — if the model already has scopes for similar filtering,
new queries belong in a model scope, not inline in the controller.
Match the project's conventions — do not generate from first principles.

Include everything in one atomic unit — migration, model, controller,
view, tests. Never split an atomic unit across commits. But never mix
categories — if a commit requires restructuring existing code, that
refactor is a separate commit before the feature commit.

**The capability list is a starting point, not a contract.** You are
closer to the code and determine commit boundaries per CLAUDE.md
conventions. You may discover a planned capability should be two or
three commits — typically because it mixes a refactor with a feature.
More commits (if each is atomic) is always correct. Split without
asking. Comment on the issue to document the deviation.

Only include code this commit uses. Factory traits, model methods, and
helper code arrive in the commit where they are first called. Do not
front-load code anticipating future commits.

**Testing mode:** Default to test-first for capabilities with clear
acceptance criteria (write the test from the spec, confirm it fails,
then implement). Use test-after for exploratory or uncertain work where
the right test isn't clear until the code exists. You own testing
decisions — the issue's testing strategy is a signal, not a directive.
Write tests only where they provide confidence. Be surgical: one precise
test proving a tricky calculation beats four tests restating what the
code obviously does.

### 3b. Verify

Before committing, verify:
- Run relevant specs
- If a migration was added, ensure it succeeds
- Lint and type checks pass (if automated via hooks, confirm no errors)

### 3c. Review

Spawn two sub-agent reviews. Each gets fresh context — the agent that
wrote the code cannot evaluate it.

Provide each reviewer with:
- The planned commit description from the issue
- What capability this commit delivers
- Instruction to run `git diff` to see the current changes
- For commits 4+, also provide the cumulative diff: `git diff main...HEAD`
  with framing: "The uncommitted `git diff` is what you are reviewing.
  The cumulative diff is ONLY for cross-commit consistency (naming,
  patterns, authorization). Findings should be about the current commit."

**Structural prompt (include in both reviews):**
"Before reviewing code quality, answer two structural questions:
(1) Does this diff contain more than one distinct change (e.g., a
refactor mixed with a feature)? If so, recommend splitting.
(2) For every new file added or pattern replaced, grep for references
to the old file/pattern. Flag any files that should be deleted."

**Pass 1 — Convention review.** Load `checklists/convention-review.md`.

**Pass 2 — Adversarial review.** Load `checklists/adversarial-review.md`.

Spawn sub-agents. Do not perform reviews inline.

**Act on feedback that:**
- Identifies correctness defects
- Flags unnecessary code or tests that don't provide confidence
- Suggests a simpler or more idiomatic approach
- Catches cross-commit consistency issues

**Ignore feedback that:**
- Adds scope beyond the plan
- Adds complexity not justified by a concrete defect
- Deviates from the agreed design in the issue

If you make changes based on feedback, re-run verification before
committing.

### 3d. Commit

Stage and commit following CLAUDE.md commit conventions. Use HEREDOC
for the message.

### 3e. Check Alignment

After committing, verify alignment with the issue's plan:
- Does the commit match what was planned?
- Were any deviations necessary?

If a deviation was needed, comment on the GitHub issue explaining what
changed and why. Leave the issue body untouched — comments make it easy
to see what diverged.

```
gh issue comment {issue-number} --body "..."
```

### 3f. Context Management

After every 3-4 commits, or when the session has been running
extensively, compact the conversation. The implementation must be
accurate — do not write code from degraded context. If context feels
polluted after debugging, `/clear` and restart the current commit
with a fresh prompt referencing the issue and git log.

### 3g. Repeat

Move to the next planned commit.

## Phase 4: Final Verification

After all planned commits are complete:

1. **Design cross-reference.** Re-read the "How it works" section (or
   behavior inventory) and verify every described behavior is covered
   by a commit. If a behavior is missing, implement it as an additional
   commit. If unsure, ask the user.
2. Run the full relevant test suite.
3. Review the commit log: `git log main..HEAD --oneline`
4. Verify the history is clean — each commit is atomic, complete,
   coherent.
5. **Final-state review.** Run `git diff main...HEAD` and spawn two
   reviewers focused ONLY on cross-cutting concerns per-commit reviews
   cannot catch:
   - Naming consistency across files
   - Authorization/scoping patterns applied inconsistently
   - Duplication that emerged across commits
   - Patterns established early that drift in later commits
   - Error handling or API call conventions used inconsistently

   **Pass 1 — Convention review.** Load `checklists/convention-review.md`.
   **Pass 2 — Adversarial review.** Load `checklists/adversarial-review.md`.

   Spawn sub-agents. Do not perform reviews inline.

   Fix issues found. Fold fixes into originating commits with
   `--fixup` + `--autosquash`.

## Phase 5: Report

Present a summary to the user:
- List of commits (`git log main..HEAD --oneline`)
- Any deviations from the plan (with references to issue comments)
- Any open items or follow-ups discovered during implementation
- Test status

Do NOT push or create a PR unless the user asks.

## Handling Mistakes

If you realize a mistake after committing:
- Fix the issue in the working tree
- Use `git commit --fixup={sha}` targeting the bad commit
- Then `GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash main` to
  fold it in cleanly (the env var bypasses the interactive editor)
- The goal is a clean history where each commit tells a clear story

Only rebase commits on this branch that haven't been pushed. If commits
have been pushed, ask the user before force-pushing.

## Failure Modes

- **Design won't work in practice:** Stop. State what's wrong and why.
  Recommend returning to /design with specific findings. Do not
  improvise around a broken design.
- **Test failures you can't resolve:** After 3-5 attempts on the same
  failure, stop. Present the error, what you've tried, and ask the user.
- **Scope creep during implementation:** If you discover work not in the
  plan, comment on the issue. Do not implement unplanned work without
  user approval.

## Rules

- Follow the plan, but trust the design. The capability list is your
  roadmap, but "How it works" is the source of truth for behavior.
- One commit at a time. Don't batch work across multiple capabilities.
- Review every commit. Convention + adversarial. No exceptions.
- Clean history matters. Fix mistakes with fixup+rebase, not "oops"
  commits.
- Comments over edits. Deviations go as issue comments. Never edit the
  issue body.
- Ask when stuck. If something doesn't work in practice, ask the user.
