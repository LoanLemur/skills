# Pipeline Handoff Evaluation: /define → /design → /execute → /review

## Summary

The pipeline is structurally sound and the primary artifact handoff (GitHub issue carrying both spec and plan) is well-designed. The critical failure is that the issue template produced by /define does not structurally match what /design and /execute look for when parsing it. Several other gaps follow from ambiguity in how artifacts are located, inconsistent checklist references, and missing handling for mid-pipeline entry.

---

## Handoff 1: /define → /design

### Broken handoff — Issue template section mismatch

/design Phase 1 instructs the agent to look for: "behaviors," "decisions," "constraints," "edge cases," "outstanding questions." The issue template (`define/assets/issue-template.md`) uses these section headers: `## Problem`, `## Goal`, `## Design` (with nested subsections: `### How it works`, `### Key decisions`, `### Constraints`, `### Edge cases`), `## Behavior Inventory`, `## Outstanding Questions`.

/design's parsing instruction says to find "Behaviors — the behavior inventory (or extract from 'How it works' if no inventory exists)." The actual header in the template is `## Behavior Inventory`, not "Behaviors." This is a minor mismatch but creates an ambiguity: an agent pattern-matching on "Behaviors" as a section header will not find it. The issue will work only because the agent reads prose, not because the structure is consistent.

More significantly, /design Phase 1 step 2 says to identify "Decisions — product-level decisions already made." The template nests key decisions under `## Design / ### Key decisions`, not as a top-level section. An agent scanning for a `## Key Decisions` header will miss it.

**Severity: Ambiguous handoff.** Likely works in practice due to prose comprehension, but structural consistency would make this reliable.

**Fix:** Either (1) align /design Phase 1's parsing language to exactly match the template headers, or (2) make the template top-level sections match what /design's parsing step enumerates.

### Lossy handoff — /define "Phase 2: Understand What Exists" findings are not in the issue

/define Phase 2 instructs extensive codebase reconnaissance: current experience, adjacent features, integration points, constraints from the existing system. The issue template has no section for these findings. The only place constraints surface is `### Constraints` under Design, which is for "Known constraints the implementation must respect" — a narrower framing.

/design Phase 2 then performs its own identical reconnaissance. This means /define's codebase exploration is discarded at the boundary. There is no mechanism for /design to benefit from /define's findings.

**Severity: Lossy handoff.** Duplicated work and risk of /design reaching different conclusions than /define did when exploring the same terrain.

**Fix:** Add a `## Codebase Context` section to the issue template (optional, omitted when empty) where /define records relevant existing patterns, files, and constraints found during Phase 2.

### Ambiguous handoff — "The issue must stand alone" vs. what /design actually needs

/define's final rule states: "The issue must stand alone. Someone running /design on this issue should have everything they need without the original conversation." /design's final rule states the same for /execute. But /design Phase 2 still requires fresh codebase exploration before proposing anything. The "stand alone" guarantee only applies to product requirements, not technical context.

This is not a bug — it's a correct division of responsibilities — but it's not stated explicitly anywhere. An agent reading both skills might conclude /design's Phase 2 exploration is redundant when the issue is well-specified.

**Severity: Ambiguous.** No functional breakage, but the stated guarantee creates a misleading implication.

**Fix:** Qualify the "must stand alone" rule in /define: "The issue must stand alone for product requirements. /design will still need to explore the codebase independently."

### Broken handoff — Missing checklist reference in /define Phase 6

/define Phase 6 specifies two review passes: "Pass 1 — Spec review. Load `checklists/spec-review.md`." and "Pass 2 — Adversarial review. Load `checklists/adversarial-review.md`."

`checklists/adversarial-review.md` exists and is correctly referenced. However, the path reference uses a relative path (`checklists/spec-review.md`, `checklists/adversarial-review.md`) without specifying that this is relative to the skills repo root. In contrast, /design Phase 2 says "Load `checklists/convention-review.md`" and /design Phase 5 references the same. /execute Phase 3c references both.

The actual checklist directory is `/Users/nick/repos/skills/checklists/`. All four skills use relative paths — which should be fine as long as the working directory is consistent — but no skill establishes what the working directory assumption is. If a skill is loaded from a different directory, all checklist paths break silently.

**Severity: Ambiguous handoff (latent broken handoff under path change).**

**Fix:** Either use absolute paths or add a note to each skill establishing the path anchor (e.g., "paths are relative to the skills repo root").

---

## Handoff 2: /design → /execute

### Broken handoff — /execute cannot distinguish the spec from the plan in the issue

/design appends the implementation plan to the existing GitHub issue. The issue now contains two distinct bodies of content:

1. The spec (/define output): Problem, Goal, Design, Behavior Inventory, Outstanding Questions
2. The plan (/design output, appended): Implementation Plan (Approach, Data model, Technical decisions, Capabilities, Behavior traceability, Implementation notes, Testing strategy)

/execute Phase 1 instructs: "Fetch the issue using `gh issue view`. Read the entire body." It then lists six things to identify: "design," "data model," "capabilities," "implementation notes," "testing strategy," "outstanding questions."

The field labeled "design" in /execute's checklist refers to "How it works" narrative, key decisions, edge cases — which comes from the spec half. The field labeled "data model" comes from the plan half. An agent must mentally parse which section of a concatenated document belongs to which conceptual layer. The two sections use similar language: the spec has `### Key decisions` (product-level) and the plan has `### Technical decisions` (implementation-level). These are easily confused.

**Severity: Ambiguous handoff.** The issue body has no structural separator between spec and plan. /execute's parsing checklist doesn't mention "look for the `## Implementation Plan` header to find the plan section."

**Fix:** Add a visible separator comment to the plan template (e.g., `---` with a label) and update /execute Phase 1 to name the specific headers to look for: "The spec is in `## Problem`, `## Goal`, `## Design`. The plan is in `## Implementation Plan`."

### Lossy handoff — Behavior traceability section is produced by /design but not consumed by /execute

/design Phase 4 produces a `### Behavior traceability` section mapping each behavior to capability numbers. /execute Phase 1 does not mention consuming this section. Its design completeness check (step 4) re-derives the behavior-to-capability mapping from scratch: "Extract every distinct behavior — each user interaction, state transition, or external integration. Verify each is covered by a capability."

This is redundant work. More importantly, /execute's re-derived mapping might differ from /design's explicit mapping, creating silent divergence.

**Severity: Lossy handoff.** The behavior traceability artifact is produced but then reconstructed instead of consumed.

**Fix:** Update /execute Phase 1 step 4 to say: "If the issue includes a `### Behavior traceability` section, use it directly as the checklist. If not, construct one from the design narrative."

### Ambiguous handoff — "The capability list is a starting point, not a contract"

Both /design Phase 4 and /execute Phase 3a use this phrase. /design says: "The executing agent determines commit boundaries per CLAUDE.md conventions — this list is a starting point, not a contract." /execute says: "The capability list is a starting point, not a contract. You are closer to the code and determine commit boundaries."

This is intentional design — /execute has authority to split capabilities. However, /execute Phase 3e says deviations should be documented as issue comments. There is no guidance in /execute about when to re-derive capability boundaries vs. when to accept the plan as-is. An agent could interpret "starting point" as license to significantly restructure the plan without any threshold for when that requires user input rather than just an issue comment.

**Severity: Ambiguous handoff.** The correct behavior is probably: minor splits (one capability → two for atomicity reasons) → comment. Major restructuring (capabilities reordered, some dropped, new ones added) → stop and ask.

**Fix:** Add a threshold to /execute Phase 3a: "If you find you need to drop or significantly reorder planned capabilities (not just split one for atomicity), stop and ask the user before proceeding."

### Missing — /execute has no branch naming convention

/execute Phase 2 says: "Use the branch name the user specifies, or ask." /design produces no branch name. There is no convention established in any skill for what branch names should look like. /review Phase 1 assumes it's reviewing "the current branch" vs. main, which works regardless of naming. But if the user is not present to specify a branch name (autonomous execution context), /execute will stop and ask — which may be intentional but is not stated as such.

**Severity: Minor gap.** No functional breakage, but missing a convention for common cases.

**Fix:** Add a default convention to /execute Phase 2, e.g., "Default to `feat/{issue-number}-{slug}` if the user doesn't specify."

---

## Handoff 3: /execute → /review

### Ambiguous handoff — /review's input parsing for issue vs. PR is complex and may fail

/review accepts an issue number, PR number, or nothing. Its detection logic: "if the argument is a URL, use the path to distinguish `/issues/` from `/pull/`. If it is a bare number, check whether a PR exists for the current branch (`gh pr view`). If one exists and its number matches, treat it as a PR. Otherwise treat it as an issue."

At the end of /execute, no PR is created: "Do NOT push or create a PR unless the user asks." So when a user invokes `/review {issue-number}` after `/execute`, the detection logic will run `gh pr view`, find no PR, and treat the bare number as an issue — which is correct. But this is not a clean handoff; it relies on the absence of a PR as an implicit signal. If the user has previously pushed and opened a PR for another branch on this repo, `gh pr view` might return that PR's number — which could match the issue number coincidentally.

**Severity: Ambiguous handoff.** The detection heuristic has a fragile edge case.

**Fix:** Make the detection explicit: "If the current branch has a PR, prefer treating a bare number as a PR number. If no PR exists for the current branch, treat it as an issue number." This is essentially what the current logic says, but stated more precisely.

### Lossy handoff — /review's behavior coverage check depends on issue being provided

/review Phase 2 performs a "Behavior coverage (completion-bias gate)" that maps behaviors to commits. This check is explicitly conditional: "If an issue was provided, cross-reference the behavior inventory against the commit list."

If the user invokes `/review` with no argument (just reviewing the current branch), behavior coverage is skipped entirely. This means the primary anti-completion-bias mechanism is optional. A user who finishes executing and says "review this" without providing the issue number gets a review without the coverage check.

**Severity: Lossy handoff.** The most important quality gate in /review is opt-in rather than default.

**Fix:** If no issue is provided, /review should attempt to infer it: check the current branch name for an issue number (e.g., `feat/608-slug` → issue 608), check recent issue comments for the branch name, or ask the user "What issue number does this branch implement? I need it for behavior coverage check."

### Lossy handoff — /review does not consume /design's behavior traceability section

/review Phase 2 maps behaviors to commits. /design already produced a `### Behavior traceability` section mapping behaviors to capabilities. /review's mapping goes directly from behaviors to commits, bypassing the intermediate capability layer.

This means /review cannot detect: "Capability N was planned but no commit implements it." It can only detect "Behavior X has no commit covering it." If /design planned a capability that doesn't directly correspond to a behavior in the inventory (e.g., a technical capability identified during design), /review has no way to check for its presence.

**Severity: Lossy handoff.** Technical capabilities added by /design are invisible to /review's coverage check.

**Fix:** /review Phase 2 should mention: "If the issue includes a `### Behavior traceability` section and a `### Capabilities` section, also verify that each planned capability has at least one covering commit."

### Ambiguous handoff — /review Phase 4 "goal-level verification" has no failure action

/review Phase 4 includes: "Re-read the issue's Problem and Goal sections. Ask: does the branch as a whole achieve the stated goal?" There is no specified action if the answer is "no." /review can fix commit structure, reword messages, and fold in fixups, but it cannot implement missing features. If goal-level verification fails, the skill has no defined failure mode.

**Severity: Ambiguous handoff.** A fundamental goal mismatch has no defined path forward.

**Fix:** Add a failure action: "If the branch does not achieve the stated goal — beyond individual uncovered behaviors — stop. Present the gap clearly and recommend returning to /execute to implement the missing work before completing review."

---

## Cross-Cutting Concerns

### Terminology mismatch — "capabilities" vs. "commits" vs. "behaviors"

The pipeline uses three related terms that are not always clearly distinguished:

- **Behaviors** (from /define) — discrete user interactions, state transitions, integration points. Defined in the Behavior Inventory.
- **Capabilities** (from /design) — the smallest independently valuable units to implement. One capability typically maps to one commit but can split.
- **Commits** (from /execute) — the actual git commits. Can differ from capabilities if /execute splits or combines.

/review's behavior coverage check maps behaviors → commits, skipping the capability layer. /design's behavior traceability maps behaviors → capabilities. /execute's design completeness check maps behaviors → capabilities. The chain is: behaviors → capabilities → commits, but no single document describes this three-layer relationship. /review only checks the first and last layers.

**Severity: Terminology mismatch + lossy handoff.** The capability layer is designed by /design but not tracked by /review.

**Fix:** Add a sentence to /review's behavior coverage check: "Note that /design may have introduced technical capabilities not in the behavior inventory. These are visible in the `### Capabilities` section of the issue. Verify each planned capability has a corresponding commit, in addition to verifying behavior coverage."

### Missing entry point — /design entered without /define output

/design's description says "Use after /define or on any issue with a behavior inventory or clear 'How it works' narrative." The skill handles this: Phase 1 step 2 says "extract from 'How it works' if no inventory exists," and step 3 says to stop and recommend /define if the issue is missing both.

This is adequately handled. The skill degrades gracefully.

### Missing entry point — /execute entered without /design output

/execute's description says "Use after /design or on any issue with a clear capability list and data model." Phase 1 step 4's design completeness check says: "If the issue includes a behavior inventory, use it as the checklist. If not, construct one from the design narrative."

This means /execute will attempt to run even without an implementation plan, constructing its own capability list from the narrative. There is no stop-and-redirect to /design the way /design has one to /define. An agent could invent capabilities without the data model, technical decisions, or behavior traceability that /design provides.

**Severity: Missing entry point guard.** /execute should stop if there is no `## Implementation Plan` section in the issue, the way /design stops if there is no behavior inventory.

**Fix:** Add to /execute Phase 1: "If the issue has no `## Implementation Plan` section, stop. Tell the user to run /design first. Do not construct a capability list from the narrative alone."

### Missing entry point — /review entered without /execute output

/review's failure mode covers "No commits on branch vs main: Stop. Confirm the branch and base." This handles the case where /review is run on a clean branch. This is adequate.

### Cross-cutting — No skill documents where checklists live relative to the project

Each skill references `checklists/` with a relative path. The `checklists/` directory is at `/Users/nick/repos/skills/checklists/`. This works when the skills repo is the working directory, but no skill states this assumption. If Claude Code loads a skill from a different working directory, all checklist `Load` instructions will silently fail.

**Severity: Latent broken handoff.** No explicit path anchor.

**Fix:** Add a note in each skill's checklist reference: "Load from the skills repo root at `checklists/{file}.md`." Alternatively, document the assumption once in a shared cross-cutting file.

### Cross-cutting — "adversarial-review.md" referenced inconsistently

/define Phase 6 references both `spec-review.md` (Pass 1) and `adversarial-review.md` (Pass 2). /design Phases 2 and 5 reference `convention-review.md` (Pass 1) and `adversarial-review.md` (Pass 2). /execute Phase 3c and Phase 4 reference `convention-review.md` (Pass 1) and `adversarial-review.md` (Pass 2).

So the checklist pairs are:
- /define: spec-review + adversarial-review
- /design: convention-review + adversarial-review
- /execute: convention-review + adversarial-review
- /review: (no external checklists — uses inline evaluation dimensions)

This is correct and intentional — spec-review is appropriate for the product spec phase, convention-review for code review phases. But /review does not use any external checklists despite having the same convention and adversarial review needs as /execute. Its evaluation dimensions are fully inline in the skill body rather than referencing the shared checklists.

**Severity: Inconsistency.** /review is the only skill that doesn't use shared checklists for its review passes, making it harder to update review standards in one place.

**Fix:** /review could reference `checklists/convention-review.md` and `checklists/adversarial-review.md` the way /execute does, or document explicitly that its inline dimensions supersede the checklists. Currently the relationship is unspecified.
