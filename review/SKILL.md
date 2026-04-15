---
name: review
description: >-
  Use when a feature branch is ready for review before merging. Use when
  the user says "review this", "look at these commits", or "check this
  branch". Also serves as the final verification gate — cross-references
  the original spec to catch completion bias. DO NOT use during
  implementation (per-commit reviews are built into /execute).
argument-hint: <issue-or-pr-number>
user-invocable: true
---

# Interactive Code Review

Review a branch interactively, commit by commit. Analyze each commit,
surface concerns, and wait for the reviewer to signal before advancing.
When changes are needed, rewrite history directly — these are private
branches and clean history is the goal.


## Input

$ARGUMENTS

Optional. Accepts any of:

- **Issue number or URL** — `608`, `#608`, or a GitHub issue URL.
  Fetch with `gh issue view` for domain context — the "How it works"
  narrative and behavior inventory inform the review but do not
  constrain it.
- **PR number or URL** — `#42` (when a PR exists for the current
  branch), or a GitHub PR URL. Fetch with `gh pr view` for title,
  description, and linked issue. If the PR links to an issue, fetch
  that too for behavior inventory context.
- **Nothing** — Review the current branch's commits vs main. No
  external context. Compensate by being more rigorous on edge case
  audit and internal consistency — without a spec to verify against,
  the code must defend itself. If the branch name contains an issue
  number (e.g., `feat/608-slug`), fetch that issue automatically.

Detection: if the argument is a URL, use the path to distinguish
`/issues/` from `/pull/`. If it is a bare number, check whether a PR
exists for the current branch (`gh pr view`). If one exists and its
number matches, treat it as a PR. Otherwise treat it as an issue.

## Review Philosophy

Every finding must pass: (1) Should this code exist at all? (2) Could
it be simpler? (3) Would deleting it improve the design? Never
recommend adding code unless there is a concrete defect. "You could
also add X" is not a finding.

## Evaluation Dimensions

Four dimensions, applied to every commit. Atomicity is the primary lens
— if commit boundaries are wrong, the other three do not matter yet.

### 1. Atomicity (primary)

The fundamental question: **What can someone do after this commit that
they could not do before?**

A commit is atomic when:
- It delivers exactly one complete capability
- Reverting it removes exactly that capability and nothing else
- It contains everything needed — migration, model, controller, view,
  tests — in one unit

Red flags:
- Layer-only commits (migration without usage, model without controller)
- Multiple capabilities in one commit
- Tests committed separately from the code they cover
- Refactors mixed with features

If a commit is not atomic, fix the structure before reviewing the code
inside it.

### 2. The 4Cs

- **Correct** — Verifiable, defect-free, handles edge cases
- **Complete** — All necessary cases covered. For every conditional,
  question the domain completeness: does the else branch handle all
  inputs? Is there a case the author assumed away? The valuable
  finding is not "this code is wrong" but "this code assumes X — is
  that always true?"
- **Concise** — Economically communicates its essential function (but
  never sacrifice clarity for brevity)
- **Clear** — Logical sequencing, coherent units, familiar structures,
  precise use of language features, follows codebase conventions per
  CLAUDE.md

### 3. Test Confidence

Only applies when the commit includes tests. If it does:

- Are tests simpler than the code they test? Complex tests are pure
  liability — you need to verify the test itself.
- Do tests cover behavior, not implementation? Test *what* happens, not
  *how*.
- Are tests at the right level? Model tests for logic, request tests
  for integration, minimal system tests.
- No mocking — this codebase uses real objects and real database calls.
- Is trivial code being tested? If the implementation is obvious, the
  test adds maintenance burden without confidence.
- Tests that do not provide confidence are harmful, not neutral. The
  bar is "does this test need to exist?"
- Are tests duplicating language or framework guarantees? Recommend
  removal.
- Are factory traits front-loaded? Traits should arrive in the commit
  where they are first used.

If the commit has no tests, evaluate whether that is correct. Obvious
code does not need tests.

### 4. Security

Brief if clean. Flag only real vulnerabilities:

- Authentication/authorization gaps
- Tenant isolation leaks (data from one merchant reaching another)
- Injection (SQL, command, XSS)
- PII exposure in logs, responses, or error messages
- Insecure direct object references
- Unbounded queries — list endpoints must apply `.limit()` or `pagy`

## Phase 1: Establish Context

Gather the raw materials:

```
git log main..HEAD --oneline
git diff main...HEAD --stat
```

If a PR was provided (or detected), fetch it:

```
gh pr view {number}
```

If an issue was provided — or the PR description links to one — fetch
it:

```
gh issue view {number}
```

Extract the behavior inventory and "How it works" narrative if present.
These inform Phase 2's structural review.

## Phase 2: Overview

Output format — terse, scannable. No flowing prose.

```
Branch: {N} commits, {X} files, +{Y}/-{Z}

{one-line arc — what this branch delivers, under 20 words}


COMMITS

- {sha} {subject}
- {sha} {subject}


STRUCTURE

- {finding} — {atomicity | ordering | narrative}
- {finding} — {atomicity | ordering | narrative}


COVERAGE

- Behavior 1 → {sha}
- Behavior 2 → {sha}, {sha}
- Behavior 3 → NOT COVERED
```

Rules:
- Omit `STRUCTURE` section entirely if clean.
- Omit `COVERAGE` section if no issue was provided.
- One line per item. No paragraphs.
- Blank line between items in multi-item sections.

Structural findings to flag (one line each):
- Subjects that do not tell a coherent story in sequence
- Subjects describing layers, not capabilities
- Wrong conventional commit prefixes
- Vague or inaccurate subjects
- Layer-only commits, multi-capability commits
- Ordering: anything depends on a later commit
- Behaviors with no covering commit (completion-bias gate)

Individual message quality (body, imperative tense) is evaluated in
Phase 3 per commit.

Present the overview. Wait for the reviewer. They may want to address
structural issues before commit-by-commit review, or say "structure
looks good, let's go."

## Phase 3: Commit Review

Work through commits sequentially, or jump to a specific commit if the
reviewer asks. Maintain a running tracker: after each commit, show
`[reviewed] sha1, sha2 | [pending] sha3, sha4`. After history surgery,
update from `git log main..HEAD --oneline`.

When the reviewer asks to revisit an already-reviewed commit ("go back
to commit 3"), clarify intent: re-review, or specific change? If
re-reviewing, present fresh. If making a change, act directly.

### For each commit

**Analyze first, then present.** Read the full diff and surrounding
code. Form an opinion before writing anything. Walk inline, evaluate
against conventions *at the moment you see the code*, and collect
findings into a severity-grouped block at the end.

**Output format:**

````
{sha} {subject}

Delivers: {one-line capability — what someone can do now}


DIFF

```diff
{chunk 1}
```
- L{n}: {one-line annotation pointing at a line}
- L{n}: {one-line annotation}

```diff
{chunk 2}
```
- L{n}: {one-line annotation}


FINDINGS

WRONG

- {Tag}: {finding} — {fix in same line, under 20 words}

- {Tag}: {finding} — {fix}


OVERCOMPLICATED

- {Tag}: {finding} — {cut/merge/replace}


STATE-SAFETY

- {finding} — {fix}


TESTS

- {finding} — {fix}


MESSAGE

- {subject issue | body issue | prefix issue}
````

Rules:
- Skip diff chunks that need no annotation.
- Annotations are one line. Point at a line number. No paragraphs.
- `DIFF` section exists only to anchor annotations. If no chunk
  needs anchoring, drop the section.
- `FINDINGS` severity sections: `WRONG`, `OVERCOMPLICATED`,
  `STATE-SAFETY`, `TESTS`, `MESSAGE`. **Omit any section with no
  findings.** Do not print empty headers.
- Each finding: one line, under 20 words, includes the fix.
- Blank line between findings. Two blank lines before section headers.
- Tags in `WRONG` / `OVERCOMPLICATED`: `Wrong`, `Missing`, `DRY`,
  `Responsibility`, `Dependency`, `Idiom`, `Purity`, `Edge-case`.
- No hedging: "not a big deal", "minor nit", "I wouldn't insist",
  "you could" are forbidden. If it is worth raising, recommend the
  fix. If not, drop it.
- No leading questions. State the answer.
- When the commit is clean, output only the header line and
  `Clean.` on the next line.

**Evaluation checklist — apply silently while reading the diff:**

- **Responsibility** — Fat models, skinny controllers. Logic belongs on
  the noun. Service objects only wrap external APIs.
- **DRY** — Same concept in two places → duplication finding.
- **Dependency** — Reaching into associations/APIs for data already
  local → flag.
- **State safety** — Crash leaves data inconsistent? Orphan state?
  Temporal coupling?
- **Error provenance** — Trace rescues/nil-checks to source. Fix at
  origin, not handler.
- **Idiom** — Rails built-ins, RESTful controllers, declarative
  patterns. Non-CRUD actions are suspect.
- **Purity** — Refactor mixed with feature → atomicity violation.
- **Edge cases** — Every conditional: what triggers the else? Every
  division: can divisor be zero? Every collection op: empty case?
  Every filter: no match case?

Cite the violated convention in the finding (`Responsibility:`,
`DRY:`, etc.). If you describe a design decision and cannot name the
convention it follows or violates, that itself is a signal — flag it.

### Sub-agent Review Passes

After inline findings, spawn two sub-agents in parallel. Each gets
the commit diff (`git show {sha}`), the behavior delivered, and
issue design context if available.

- **Sub-agent 1 (Convention).** Loads
  `checklists/convention-review.md`.
- **Sub-agent 2 (Adversarial).** Loads
  `checklists/adversarial-review.md`.

Sub-agent output format requirement (embed in their prompts):

```
WRONG
- {Tag}: {finding under 20 words with fix}

OVERCOMPLICATED
- {Tag}: {finding with fix}

STATE-SAFETY
- {finding with fix}

Omit empty sections. One line per finding. No prose.
```

Verify every specific claim (file:line, API name, quoted text) before
including it. Drop unverifiable claims. Never deflate a verified
WRONG to a softer category.

**Merge sub-agent findings into the same severity-grouped block as
inline findings — do not present them as separate prose sections.**
Tag the source inline when it matters:

```
FINDINGS (inline + sub-agents)

WRONG

- State-safety: `update!` after Stripe charge → orphan on raise.
  Call Stripe first, record after. — adversarial

- DRY: `fee_cents` duplicates `Merchant#fee_cents`. Delegate. —
  convention


OVERCOMPLICATED

- Cut: `format_amount` wraps `number_to_currency` with no change.
  Call it directly. — inline
```

Source tags: `— inline`, `— convention`, `— adversarial`, or a
combination (`— inline, convention`) when multiple agents surface the
same issue. Merge duplicates, don't repeat.

**Tone.**

- State findings as recommendations with the fix.
- Clean commits get one word: `Clean.` — do not manufacture findings.
- No sycophancy: no "great", "nice", "good catch", "I agree."
- Positive acknowledgment of good patterns is allowed only when the
  pattern is non-obvious and worth naming. One line, no superlatives:
  `limits_concurrency here prevents double-billing on retries.`
- No narration: "I see why they did this" → cut.
- No wrap-up sentences: "Overall this commit looks solid" → cut.
- No "Next?" or "Shall we move on?" — reviewer advances.

**Wait for the reviewer.** They may:
- Ask questions
- Agree and ask you to fix
- Disagree and explain
- Say "next"

Do not advance until the reviewer signals.

### Making Changes

When the reviewer agrees something should change, rewrite history
directly. These are private branches — clean history is the goal.

Before any history surgery: `git log --oneline origin/main..HEAD
2>/dev/null`. If pushed, warn: "This branch has been pushed.
Rewriting history will require a force-push."

Operations:

- **Fixup** — `--fixup` + `--autosquash`
- **Reword** — `GIT_SEQUENCE_EDITOR` + `GIT_EDITOR`
- **Squash** — `reset --soft` (adjacent) or `GIT_SEQUENCE_EDITOR`
  (non-adjacent)
- **Split** — mark `edit`, reset, selectively re-commit
- **Reorder** — `GIT_SEQUENCE_EDITOR` script
- **Drop** — remove entirely

All operations use `GIT_SEQUENCE_EDITOR` and `GIT_EDITOR` for
non-interactive `git rebase -i`. On macOS, `sed -i ''`.

### Handling Conflicts

Record HEAD before any rebase: `old_head=$(git rev-parse HEAD)`

Resolve mechanical conflicts directly (renames propagating, later
version clearly correct). Present ambiguous conflicts (competing
design choices) to the reviewer.

After any rebase: `git range-diff main $old_head HEAD`. Flag commits
with non-trivial content changes when reviewing them.

**Verify commit messages after every rebase.** Surgery changes
content but not messages. Reword proactively when they drift.

**Trace defects to their origin commit.** Fix where introduced.
`git log --oneline main..HEAD -- {file}` to trace.

## Phase 4: Wrap-up

After all commits have been reviewed, output:

```
FINAL LOG

{output of git log main..HEAD --oneline}


CHANGES

- {fixup | reword | split | squash | drop | reorder}: {sha} — {what}
- {fixup | reword | split | squash | drop | reorder}: {sha} — {what}


COVERAGE (re-check)

- Behavior 1 → {sha}
- Behavior 2 → {sha}
- Behavior 3 → NOT COVERED {flag}


GOAL

- {met | not met} — {one-line reason if not met}


CROSS-COMMIT

- {finding} — {pattern drift | naming | duplication | inconsistency}


OPEN

- {follow-up}


CHECKS

- tests: {pass | fail — N failures}
- lint: {pass | fail}
- types: {pass | fail}
```

Rules:
- Omit any section with no entries. If `CHANGES` is empty because
  the branch was clean, drop the section.
- `GOAL` section only appears if an issue was provided. "Met" is
  enough — no explanation needed.
- `COVERAGE` section only appears if an issue was provided.
- If `GOAL` is `not met`, stop. Recommend returning to `/execute`.
- Run the project's test suite, linter, type checker from CLAUDE.md
  or CI config. Fix failures. Fold fixes with `--fixup` +
  `--autosquash`. Note any such fix under `CHANGES`.

**Narrative re-check.** Re-read `git log main..HEAD --oneline`.
History surgery may have changed the arc. Apply Phase 2 structural
evaluation. Reword if needed — note under `CHANGES`.

**Cross-commit pass.** Scan for issues only visible across the
branch:
- Patterns established early that drift later
- Naming introduced but not followed consistently
- Duplication across commits suggesting a missing abstraction
- Authorization / error handling / API call conventions used
  inconsistently

One line per finding. Omit section if clean.

## Failure Modes

- **No commits on branch vs main:** Stop. Confirm branch and base.
- **Issue/PR not found:** Proceed without external context. Note
  that behavior coverage cannot be checked.
- **Unrecoverable rebase failure:** `git rebase --abort`, report,
  let reviewer decide.
- **Uncommitted changes on branch:** Stop. Ask reviewer to commit
  or stash.

## Rules

- **Atomicity is the primary lens.** Fix structure before reviewing
  code.
- **Evaluate, do not narrate.** Cite the convention at the moment
  of description.
- **No changes without agreement.** Discuss before acting.
- **Code is liability.** "Does this need to exist?" not "does this
  hurt?"
- **Recommend, do not observe.** Every finding includes the fix.
- **Go to the cleanest answer.** No half-measures.
- **Fresh perspective.** Review as if first time.

## Output Discipline

Applies to every phase's output.

- **One finding per bullet.** No paragraphs.
- **Under 20 words per finding.** Includes the fix.
- **Blank line between findings.** Two blank lines before section
  headers. Scannability is the point.
- **Omit empty sections entirely.** Never print a header with nothing
  under it. Never write "No findings" under a section — drop the
  section.
- **Clean commits get one word: `Clean.`** Do not manufacture
  findings to fill space.
- **No preamble, no wrap-up.** No "Here is my review of commit X",
  no "Overall this looks solid", no "Let me know if you want to
  discuss." Start with the finding. End when findings end.
- **No hedging.** Forbidden: "might", "could be", "perhaps",
  "consider", "not a big deal", "minor nit", "I wouldn't insist",
  "worth discussing."
- **No leading questions.** State the answer.
- **Short sentences.** Fragments acceptable when unambiguous.
- **Per-commit review fits on one screen** (roughly 40 lines). If
  it doesn't, you are narrating. Cut.
