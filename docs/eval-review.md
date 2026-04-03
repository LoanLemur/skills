# Evaluation: /review SKILL.md

Evaluated against: `checklists/spec-review.md`, `checklists/convention-review.md`, `checklists/adversarial-review.md`, `docs/24-skill-writing-craft.md`, `docs/28-skill-map.md`, `docs/00-synthesis.md`.

Line count: 497 (passes the 500-line limit, barely).

---

## 1. Broken — Agent would produce wrong output or get stuck

**B1. Sub-agents are never spawned (design gap vs. skill-map spec)**

`docs/28-skill-map.md` lines 89-93 is explicit: the review mechanism is two checklist-based sub-agents — `convention-reviewer` and `adversarial-reviewer` — with their checklists living in `references/convention-review-checklist.md` and `references/adversarial-review-checklist.md`. The skill has no `references/` directory, no sub-agent spawning instruction, and no reference to either checklist file. Instead, a partial version of the convention checklist is inlined at SKILL.md lines 287-320. An agent following SKILL.md performs a single-context, inline review and never achieves the context-isolated passes the skill-map mandates. The adversarial pass is missing entirely.

**B2. "No changes without agreement" contradicts "Resolve conflicts directly"**

Rules section (line 481): "No changes without agreement. Discuss findings before acting."

Handling Conflicts section (lines 391-393): "Resolve conflicts directly. Mechanical conflicts (renames propagating, later version clearly correct) resolve without asking."

These are irreconcilable during a rebase. When the agent executes a squash or fixup (authorized by the reviewer) and hits a mechanical conflict, rule B says stop and discuss; the Handling Conflicts section says proceed. An agent without a tiebreaker will apply whichever of these it encounters last — which is arbitrary.

Fix: add one sentence to Handling Conflicts clarifying that direct conflict resolution is an exception to the agreement requirement, scoped to conflicts arising from already-approved operations.

**B3. Inline convention checklist omits half of the items in the actual checklist**

SKILL.md lines 287-320 include 7 convention items (Responsibility, DRY, Dependency, State safety, Error provenance, Idiom conformance, Commit purity). The actual `checklists/convention-review.md` also contains: HTTP semantics (GET must be safe, mutation verbs), Hotwire idioms (Turbo Frames/Streams), convention conformance in new files (compare against existing templates), security/tenant isolation (authorize on every action, scoped queries, strong parameters, no PII in URLs), and test honesty (`it` block descriptions must match assertions). An agent following SKILL.md would silently skip these entire categories on every commit.

**B4. "Service objects" cited as acceptable in the inline checklist, contradicting CLAUDE.md**

SKILL.md line 291: "Business logic in models or service objects, not controllers."

CLAUDE.md (global instructions): "Verb-named classes with a `#call` method are the Command pattern — they separate behavior from data and create anemic models... The only acceptable 'service' is a wrapper around an external API boundary."

An agent following the inline checklist would tell the reviewer that a `ProcessPaymentService` class is fine. CLAUDE.md says that finding should be flagged as a convention violation, not praised as correct placement. This is a direct contradiction that produces wrong review output.

---

## 2. Unclear — Agent might interpret this multiple ways

**U1. Description opens with a process summary, not a trigger**

SKILL.md lines 3-4: "Interactive commit-by-commit code review with history surgery."

`docs/00-synthesis.md` (The Description Field Is Make-or-Break): "Descriptions must contain only triggering conditions ('Use when...'), never process summaries. Descriptions that summarize the workflow cause Claude to follow the summary and skip the skill body entirely."

The description leads with a capability summary, which is the exact failure mode documented. The triggering conditions follow on lines 5-9 and are well-formed, but the summary sentence shouldn't be there. This is labeled a "critical rule" in the source.

**U2. "Track which commits have been reviewed" has no mechanism**

Line 211: "Track which commits have been reviewed and which are pending."

No instruction on how. Does the agent maintain a mental list? Output a running tracker visible to the user after each commit? Write a file? The phrase is imperative but the implementation is completely open. In a long-reviewed branch with history surgery (commits renumbered mid-session), an agent with only a mental list will lose track.

**U3. Phase 2 gate only enumerates two reviewer responses**

Lines 203-206: "Present the overview and structural findings. Wait for the reviewer. They may want to address structural issues... Or they may say 'structure looks good, let's go commit by commit.'"

Two paths are named. Missing: the reviewer asks a clarifying question, the reviewer disagrees with a structural finding, the reviewer wants to fix only some structural issues and move on, the reviewer is silent. An agent that only knows two valid next states will either stall or pick one arbitrarily.

**U4. "Curious, not adversarial: 'Why X over Y?'" contradicts "Leading questions are a cop-out"**

Lines 328-329 (tone guidance): "Curious, not adversarial: 'Why X over Y?' not 'You should have done Y'"

Lines 352-353 (findings guidance): "Leading questions are a cop-out. 'Is this more complex than it needs to be?' puts the analytical burden on the reviewer. State the answer."

"Why X over Y?" is a leading question. The tone section teaches the agent to ask it; the findings section prohibits asking it. An agent will apply whichever section it hits more recently in the context, inconsistently.

**U5. "Trace defects to their origin commit" doesn't say what to do next**

Lines 402-403: "Trace defects to their origin commit. Fix where introduced, not where noticed."

If the origin commit was already presented and approved by the reviewer in Phase 3, what does the agent do? Go back and re-present it? Apply a fixup silently? Ask the reviewer to re-open it? The instruction stops at "fix where introduced" without handling the already-reviewed case.

**U6. `{relevant specs}` in the final state command is unresolved**

Line 443: `bundle exec rspec {relevant specs}`

The placeholder is never defined. How does the agent determine which specs are relevant? Run all specs? Run specs matching files changed in the branch diff? The other two final-state commands (`rubocop --parallel`, `erb_lint --lint-all`) are complete; this one is a template fragment.

---

## 3. Missing — Scenarios the skill doesn't handle

**M1. Reviewer makes manual edits mid-review**

No instruction for when the reviewer directly edits code, changes a file, or amends a commit during Phase 3 without asking the agent to do it. The agent's diff-based understanding of the commit is now stale. The skill should say: re-read the affected commit's diff before continuing if the reviewer indicates they've made manual changes.

**M2. Branch has already been pushed / is not fully private**

The skill states "These are private branches" twice (lines 18, 367) and authorizes free history surgery. No instruction on what to do if the reviewer mentions the branch has been pushed to a shared remote, has a PR with comments, or has been reviewed by others. Force-pushing over a shared PR is destructive. The skill should have a failure mode or at minimum a check: "If the branch has been pushed and others may have pulled it, warn before any rebase."

**M3. Long-branch handling is underspecified**

Line 166: "Group related commits into chapters if the branch is long."

"Long" is undefined. What triggers chapter grouping — 5 commits? 10? 20? The skill says to group related commits into chapters for the narrative but gives no guidance on whether chapters affect Phase 3 sequencing (does the agent do Phase 3 chapter by chapter with waits between chapters?) or whether the reviewer approves the chapter grouping before diving in.

**M4. No re-verification after history surgery in Phase 3**

After the agent executes a squash, split, fixup, or reorder, it continues reviewing the next commit. But the rebase may have introduced content changes into commits that haven't been reviewed yet. Lines 395-396 say to run `git range-diff` after rebase to find non-trivial changes — but this instruction is in "Handling Conflicts," scoped to conflict resolution. There is no instruction to run `range-diff` after every surgery operation to verify the rewrite didn't silently alter reviewed content.

**M5. PR description quality is not evaluated**

Phase 1 fetches the PR with `gh pr view`. Phase 4 does a final narrative check on commit messages. But neither phase evaluates the PR description itself — whether it accurately describes what the branch does, whether it links to the right issue, whether it would give a reviewer sufficient context. The skill fetches the PR but only extracts it for issue-linking context, not as an artifact to review.

---

## 4. Convention violations — wouldn't cause failure but don't follow craft conventions

**CV1. Final state commands are Rails-specific and hardcoded**

Lines 440-443:
```
bundle exec rubocop --parallel
bundle exec erb_lint --lint-all
bundle exec rspec {relevant specs}
```

These commands assume a Ruby on Rails project with Rubocop and erb_lint installed. A portable skill should not hardcode project-specific tooling. Per `docs/24-skill-writing-craft.md`: include only what Claude doesn't already know, and don't include things that belong in project-level CLAUDE.md. These commands belong in the project's CLAUDE.md, not a shared skill.

**CV2. Rules section is a repetition of the body**

Lines 467-496 restate 14 rules that are already expressed as behavior in the phase instructions: "Analyze before presenting" (Phase 3, line 222), "Wait for the reviewer" (Phase 3, line 355), "No changes without agreement" (Phase 3, line 481 — also stated in Making Changes section), etc. The Rules section adds no new information. Per `docs/24-skill-writing-craft.md` on instruction budget: every instruction dilutes every other, and repetition is pure cost.

**CV3. Tone example teaches sycophantic language**

Line 326: `"Good call using \`limits_concurrency\` here — prevents double-billing"`

This is an example of positive tone the agent should adopt. CLAUDE.md's global instructions prohibit "Good call" as sycophancy. The skill is teaching the agent to produce output that CLAUDE.md forbids. The example should be rewritten to illustrate positive acknowledgment without the sycophantic opener: `"\`limits_concurrency\` is the right choice here — prevents double-billing on concurrent retries."`

**CV4. Review Philosophy section is explanation rather than direction**

Lines 44-56 explain the philosophy behind the review approach. Per `docs/24-skill-writing-craft.md`: "Lead with the directive. Add explanation only if it changes behavior." The three philosophy questions (Should this code exist? Could it be simpler? Would deleting it improve the design?) are valid, but they're framed as a reasoning framework rather than imperatives. They could be tightened to three bullet directives without the preamble.

---

## 5. Nitpicks

**N1. Argument detection logic doesn't handle `gh issue view` failure**

Lines 39-43 handle URL vs. bare number detection but don't say what to do if a bare number is provided, `gh pr view` returns no PR, and `gh issue view {number}` also fails (the issue doesn't exist, wrong repo, auth problem). The skill says "treat it as an issue" but has no fallback when that also fails.

**N2. Git operation list omits actual `GIT_SEQUENCE_EDITOR` script syntax**

Lines 370-380 list six available operations. The "Reorder" entry says `GIT_SEQUENCE_EDITOR` script, and the footnote at lines 382-384 says all operations use `GIT_SEQUENCE_EDITOR` to control `git rebase -i` non-interactively. No example of what that script looks like is provided. An agent that hasn't done this before will have to invent it. One example (e.g., for reword) would eliminate ambiguity at low cost.

**N3. `git diff main...HEAD --stat` uses three-dot syntax without explanation**

Line 143: `git diff main...HEAD --stat`

Three-dot diff (symmetric difference) vs. two-dot diff (range) produce different output. The skill uses three-dot (shows changes relative to merge base, which is usually what you want for branch review) but doesn't explain why. Most developers have only seen two-dot. Not a blocker — but an agent explaining this to a reviewer would look wrong if it doesn't understand the distinction.

**N4. "Rewrite history freely" and "No changes without agreement" are both in Rules**

Lines 482 and 484 appear adjacent in the Rules list: "No changes without agreement" and "Rewrite history freely." The juxtaposition without any ordering or scoping explanation is confusing in isolation — one says ask first, the other says act. The body of the skill handles this correctly (get agreement, then rewrite), but the Rules section doesn't preserve that sequencing.

---

## Goal-level check

The skill-map says /review's job is: "Code review with history surgery + goal verification" with output: "Clean commit history, verification report (behavior inventory cross-referenced against commits)."

The skill delivers the first part (history surgery, per-commit walkthrough) well. The verification report output is partially there — Phase 4 includes a behavior coverage re-check and goal-level verification. But the skill never specifies what a "verification report" looks like as an output artifact. Does the agent produce a written document? A structured summary? A pass/fail verdict? The summary at lines 449-455 lists four bullet points but doesn't format them as a report the user could hand to a reviewer or attach to a PR.

The skill-map's key enhancement for /review was "Merge /finish's spec cross-referencing into this skill, rule-compliance checklist derived from CLAUDE.md, selective context transfer (spec + checklist, not conversation history)." The selective context transfer mechanism (spawning sub-agents with isolated context) is entirely absent. This is the most significant gap relative to what the skill-map says this skill should do.
