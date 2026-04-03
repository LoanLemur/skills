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
  external context; the code speaks for itself. Without an issue,
  behavior coverage cannot be checked. If the branch name contains an
  issue number (e.g., `feat/608-slug`), fetch that issue automatically
  for coverage context.

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
- Code is liability, not asset. Tests are code. Tests that do not
  provide confidence are not neutral — they are harmful. The bar is
  "does this test need to exist?" not "does this test hurt?"
- Are tests duplicating language or framework guarantees? A test that
  verifies Ruby can do integer division, or that Rails validates
  presence, provides zero confidence. Recommend removal.
- Are factory traits or test setup front-loaded? Traits should arrive
  in the commit where they are first used. Traits for untested states
  are dead code.

If the commit has no tests, evaluate whether that is correct. Tests
provide confidence only when the code has non-trivial logic. Obvious
code does not need tests.

### 4. Security

Brief if clean. Flag only real vulnerabilities:

- Authentication/authorization gaps
- Tenant isolation leaks (data from one merchant reaching another)
- Injection (SQL, command, XSS)
- PII exposure in logs, responses, or error messages
- Insecure direct object references
- Unbounded queries — search, autocomplete, and list endpoints must
  apply `.limit()` or use `pagy`. Missing limits enable enumeration
  and performance degradation.

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

Present the branch as a narrative — not a list of commits, but a story.
What does this branch accomplish? What is the arc? Group related commits
into chapters if the branch is long.

Then evaluate the **commit structure** before looking at any code:

**Commit narrative** — Read `git log main..HEAD --oneline`. Do the
subjects alone tell a coherent story? Could someone understand what this
branch does from the one-line log? Flag narrative-level issues:
- Subjects that do not form a coherent story when read in sequence
- Subjects that describe layers instead of capabilities
- Wrong conventional commit prefixes
- Vague or inaccurate subjects

Individual message quality (body content, imperative tense, length) is
evaluated per-commit in Phase 3.

**Atomicity scan** — Does each commit deliver exactly one capability?
Flag layer-only commits, multi-capability commits, or commits that
should be merged with adjacent ones.

**Ordering** — Would the narrative be clearer if commits were
reordered? Does anything depend on something introduced in a later
commit?

**Behavior coverage (completion-bias gate)** — If an issue was provided,
cross-reference the behavior inventory against the commit list. Present
a visible mapping:

```
Behavior 1 → Commit {sha short}
Behavior 2 → Commits {sha}, {sha}
Behavior 3 → NOT COVERED
```

Any behaviors not covered by any commit? Any commits that do not map to
a behavior? This is the primary gate against completion bias — the
implementation agent may have declared "done" without covering everything.

If the issue includes a `### Capabilities` section, also verify each
planned capability has at least one covering commit.

Present the overview and structural findings. Wait for the reviewer.
They may want to address structural issues (reorder, split, squash,
reword) before diving into code-level review. Or they may say "structure
looks good, let's go commit by commit."

## Phase 3: Commit Review

Work through commits sequentially, or jump to a specific commit if the
reviewer asks. Maintain a running tracker: after each commit review,
show `[reviewed] sha1, sha2 | [pending] sha3, sha4`. After history
surgery, update the list from `git log main..HEAD --oneline`.

When the reviewer asks to revisit an already-reviewed commit (e.g.,
"go back to commit 3"), clarify intent: are they re-reviewing it, or
do they want to make a specific change? If re-reviewing, present the
commit fresh. If making a change, go directly to the action.

### For each commit

**Analyze first, then present.** Read the full diff and surrounding
code. Form an opinion before presenting anything. The walkthrough should
have the analysis already baked in — not "here is the diff, and here is
what I think" but a single annotated narrative.

**Present as an annotated diff walkthrough:**

Open with what capability this commit delivers (one sentence). Then
walk through the diff itself — show the actual code in ```diff blocks,
interleaved with commentary. Findings are anchored to the exact lines
they reference, not listed separately.

Break the diff into logical chunks with interleaved commentary. Skip
chunks that need no comment. Example:

````
```diff
+  def charge!
+    return false unless pending?
```

Good guard — prevents double-charging on retry.

```diff
+    stripe_charge = Stripe::Charge.create(amount: amount_cents)
+    update!(status: :completed, stripe_charge_id: stripe_charge.id)
```

**State safety**: If `update!` raises after Stripe charges, the
charge is orphaned. Call Stripe first, then record the result.
````

Explain non-obvious patterns and framework magic inline, as a peer.
Evaluate the commit message at the end: accurate subject, imperative,
under 50 chars, correct prefix, body explains why not what. Apply
the four evaluation dimensions throughout.

**Evaluate design decisions against conventions — inline, not after.**

The failure mode is narration bias — accurately describing what code
does without judging whether it should do it that way. Comprehension
masquerades as approval. "I see why they did this" silently becomes
"this is fine."

Guard against this: at the moment you describe a design decision in
the walkthrough, cite the CLAUDE.md convention it follows or violates.
The walkthrough is the input to evaluation, not the output of the
review. Examples:

- "The job orchestrates the charge" → fat models says logic belongs
  on the model. That is a finding.
- "Creates the record as pending, then calls the API" → what state
  is this record in if the API call crashes?
- "Extracts a partial and adds the edit form" → refactors mixed
  with features is an atomicity violation.

When you encounter a design decision and cannot name the convention it
follows, that is itself a signal worth flagging.

**Convention evaluation checklist** — apply at the moment of
description, not in a separate pass. This inline checklist guides the
walkthrough. The full convention and adversarial checklists are applied
by sub-agents after the walkthrough.

- **Responsibility** — Does the logic belong where it is? Fat models,
  skinny controllers. Business logic in models, not controllers. Service
  objects are acceptable only as wrappers around external API boundaries.
  If a model already has a method that does this, the controller should
  not reimplement it. Does the controller export ivars the view can
  reach through an association on another ivar?
- **DRY** — Is the same concept (method name, calculation,
  responsibility) implemented in more than one place? When two models
  or two files implement the same thing, flag it. Delegation is
  usually the fix. Frame it as duplication, not as a correctness
  question.
- **Dependency** — When code reaches into an association, API, or
  external service, ask: is this value already available locally? A
  method that calls `subscription.status` when the model has a
  `status` column creates unnecessary coupling and potentially an
  API call. Prefer local state over reaching into dependencies.
- **State safety** — Would a crash leave data inconsistent? Are there
  crash-safe defaults? Does a public method silently depend on another
  method being called first (temporal/sequential coupling)?
- **Error provenance** — When you see error handling (rescue, nil-
  check, guard clause), trace it to its source. Where does the error
  originate? Would changing the error at the source (e.g., raising a
  typed exception instead of returning nil) eliminate all these
  downstream handlers? The individual handler may be fine; the pattern
  may be wrong.
- **Idiom conformance** — Does code use Rails built-ins? RESTful
  controllers? Declarative over imperative? `head :ok` not
  `render json: nil`. Any non-CRUD controller action is suspect — can
  it be a parameter to an existing CRUD action or a new sub-resource
  controller?
- **Commit purity** — Does the commit mix a refactor with a feature?
  Would it belong in a separate commit?

**Tone: senior co-owner.**

- Direct on real issues: "This has a race condition because X"
- Brief on minor issues: "Nitpick: trailing whitespace on line 42"
- Positive on good patterns: "`limits_concurrency` is the right choice
  here — prevents double-billing on concurrent retries"
- Curious, not adversarial: "X is simpler than Y because Z — worth
  switching?" not "You should have done Y"
- When the commit is clean, say so. Do not manufacture findings.

**Findings are recommendations, not observations.**

State findings as recommendations with the best fix, not as open
questions for the reviewer to solve. "This should use BigDecimal —
float multiplication loses precision on currency" not "you could use
BigDecimal here if you wanted." The reviewer can disagree, but they
should not have to drag the conclusion out of you.

- Go to the cleanest answer the first time. If a test assertion
  should be simpler, recommend the simplest correct version — do not
  stop at an intermediate improvement that still has the same problem.
- Do not soften real findings. "Not a big deal" and "I wouldn't
  insist" signal that you do not believe your own analysis. If it is
  worth raising, recommend the fix. If it is not worth raising, do
  not mention it.
- Never end a finding with "Next?" or "Shall we move on?" — the
  reviewer decides when to advance. Presenting a finding and
  immediately offering to skip it undermines the finding.
- Leading questions are a cop-out. "Is this more complex than it
  needs to be?" puts the analytical burden on the reviewer. State
  the answer: "This is more complex than it needs to be — X would
  be simpler because Y."

**Wait for the reviewer.** They may:

- Ask questions about the code or decisions
- Agree with a finding and ask you to fix it
- Disagree with a finding and explain why
- Say "looks good, next"

Do not advance to the next commit until the reviewer signals.

### Sub-agent Review Passes

After the inline walkthrough and findings for each commit, spawn two
sub-agent reviews. Provide each with the commit diff (`git show {sha}`),
the behavior this commit delivers, and the issue's design context if
available.

**Sub-agent 1 (Convention).** Load `checklists/convention-review.md`
from the skills repository root.

**Sub-agent 2 (Adversarial).** Load `checklists/adversarial-review.md`
from the skills repository root.

The sub-agents catch what inline review misses due to context
familiarity. Present their findings alongside yours before waiting for
the reviewer.

### Making Changes

When the reviewer agrees something should change, rewrite history
directly. These are private branches — clean history is the goal.

Before any history surgery, check:
`git log --oneline origin/main..HEAD 2>/dev/null`. If the branch has
been pushed to a remote, warn the reviewer before any rebase: "This
branch has been pushed. Rewriting history will require a force-push."

Available operations:

- **Fixup** — Fold a small fix into an earlier commit (`--fixup` +
  `--autosquash`)
- **Reword** — Fix a commit message (`GIT_SEQUENCE_EDITOR` +
  `GIT_EDITOR`)
- **Squash** — Combine commits (`reset --soft` for adjacent,
  `GIT_SEQUENCE_EDITOR` for non-adjacent)
- **Split** — Break one commit into multiple (mark as `edit`, reset,
  selectively re-commit)
- **Reorder** — Change commit sequence (`GIT_SEQUENCE_EDITOR` script)
- **Drop** — Remove a commit entirely

All operations use `GIT_SEQUENCE_EDITOR` and `GIT_EDITOR` to control
`git rebase -i` non-interactively. On macOS, `sed -i` requires an
empty string argument (`sed -i ''`).

### Handling Conflicts

Record HEAD before any rebase: `old_head=$(git rev-parse HEAD)`

Resolve conflicts directly. Mechanical conflicts (renames propagating,
later version clearly correct) resolve without asking. Genuinely
ambiguous conflicts (competing design choices) — present both options
to the reviewer. Direct conflict resolution during already-approved
operations is an exception to the "no changes without agreement" rule.

After any rebase, run `git range-diff main $old_head HEAD` to verify
the rewrite didn't alter content in unreviewed commits. Flag commits
with non-trivial content changes when reviewing them.

**Verify commit messages after every rebase.** Surgery changes content
but not messages. Reword proactively when content no longer matches.

**Trace defects to their origin commit.** Fix where introduced, not
where noticed. Use `git log --oneline main..HEAD -- {file}` to trace.

## Phase 4: Wrap-up

After all commits have been reviewed:

**Final narrative check.** Re-read `git log main..HEAD --oneline`.
History surgery during Phase 3 — splits, squashes, rewords, reorders
— may have changed the arc. Apply the same evaluation as Phase 2: do
the subjects form a coherent story when read in sequence? Are prefixes
still correct? Reword if not.

**Behavior coverage re-check.** If an issue was provided with a
behavior inventory, re-run the mapping from Phase 2 against the final
commit list. Squashes or drops during Phase 3 may have removed coverage.
If any behavior is no longer addressed, flag it. Present the updated
mapping to the user.

**Goal-level verification.** Re-read the issue's Problem and Goal
sections. Ask: does the branch as a whole achieve the stated goal? The
behavior inventory may be complete but the goal not met if behaviors
were interpreted narrowly. This is the final check against the original
intent, not just the task list. If the branch does not achieve the
stated goal, stop. Present the gap and recommend returning to /execute
to implement missing work.

**Cross-commit consistency pass.** Look for issues that only become
visible across the full branch — not within any single commit:

- Patterns established early that drift or contradict in later commits
- Naming conventions introduced but not followed consistently
- Duplication across commits that suggests a missing shared abstraction
- Authorization, error handling, or API call conventions used
  inconsistently across files

**Final state:**

```
git log main..HEAD --oneline
```

Run the project's test suite, linter, and type checker. Use commands
from CLAUDE.md or the project's CI configuration. Fix any failures.
Fold fixes into the originating commit with `--fixup` + `--autosquash`.
If a fix touches code reviewed in Phase 3, note the change in the
summary.

**Summary:**

- The final commit log — the story the branch now tells
- Changes made during review (fixups, rewords, reorders, splits,
  squashes, drops)
- Any open items or follow-ups noted during review
- Test and lint status

## Failure Modes

- **No commits on branch vs main:** Stop. Confirm the branch and base.
- **Issue/PR not found:** Proceed without external context. The code
  speaks for itself — but note that behavior coverage cannot be checked.
- **Unrecoverable rebase failure:** `git rebase --abort`, report what
  happened, let the reviewer decide next steps.
- **Uncommitted changes on branch:** Stop. Ask the reviewer to commit
  or stash before starting.

## Rules

- **Atomicity is the primary lens.** If commit boundaries are wrong,
  fix structure before reviewing code.
- **Evaluate, do not narrate.** Describing code accurately is not
  reviewing it. Cite the convention at the moment of description.
- **No changes without agreement.** Discuss findings before acting.
- **Code is liability.** The question is "does this need to exist?"
  not "does this hurt?"
- **Recommend, do not observe.** Every finding includes the fix.
- **Go to the cleanest answer.** Do not stop at half-measures.
- **Fresh perspective.** Review as if seeing this code for the first
  time. Do not assume prior reviews caught everything.
