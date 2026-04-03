# Evaluation: /design Skill

Evaluated against: `design/SKILL.md` (271 lines), `design/assets/plan-template.md`, `checklists/convention-review.md`, `checklists/adversarial-review.md`, `docs/24-skill-writing-craft.md`, `docs/28-skill-map.md`, `docs/00-synthesis.md`.

---

## 1. Broken — Agent Would Produce Wrong Output or Get Stuck

### 1.1 Direct Contradiction Between Skill and Skill-Map Goal

`28-skill-map.md` line 67 states as a key enhancement for /design:

> "Must structurally produce competing approaches before selecting one (anti-myopia, not advisory)"

`SKILL.md` Phase 3 (lines 88-95) says the opposite:

> "When there's a clear best approach, present it directly. Don't manufacture alternatives — a strawman comparison wastes time and insults the reader."

These are mutually exclusive. The skill-map's intent was that the agent always generates alternatives before selecting one — the "not advisory" qualifier means it was supposed to be a hard structural requirement, not a suggestion. The skill as written leaves the decision entirely to the agent, which will nearly always find a "clear best approach" and skip adversarial comparison. If the skill-map is the goal spec, Phase 3 must be rewritten to require at least two approaches every time. If the skill's current Phase 3 is correct, the skill-map must be updated.

### 1.2 Checklist Paths Are Unresolvable

Phase 2 (line 69) and Phase 5 (line 192) instruct the agent to load `checklists/convention-review.md` and `checklists/adversarial-review.md`. These files live at `/skills/checklists/` — at the repo root, not inside `design/`. The skill gives no base path. An agent reading this skill in isolation cannot determine how to resolve `checklists/` relative to the skill's own location at `design/SKILL.md`. It will either guess wrong or fail silently.

Paths should be anchored: either absolute from repo root (`checklists/convention-review.md` with an instruction that paths are repo-root-relative) or the skill should say "load using `Read` from the path the user provides at session start." Progressive disclosure requires SKILL.md link directly to every resource — unresolvable paths break this.

### 1.3 Phase 2 Sub-Agents Are Asked to Review an Architecture That Doesn't Exist Yet

Phase 2 (lines 64-79) spawns two sub-agent reviews before Phase 3 proposes any approach. The convention-review checklist is written for code diff review — it begins with "Run `git diff`" and checks Hotwire idioms, Rails patterns, dead code, and commit purity. None of this applies pre-design.

The sub-agents in Phase 2 receive: behavior inventory, relevant files list, constraints. They receive no proposed approach because none exists. The instruction tells the convention sub-agent to "assess whether there's one obvious idiomatic approach or genuinely different ways to do this" — but this is exactly what Phase 3 does. The Phase 2 round duplicates Phase 3 reasoning without the benefit of the agent's own codebase exploration, and uses a checklist that does not fit the task. An agent following this literally would spawn sub-agents, receive inconclusive or misapplied output, and proceed to Phase 3 regardless — making Phase 2 reviews performative.

---

## 2. Unclear — Agent Might Interpret Multiple Ways

### 2.1 Phase 2 Has No Gate

Phase 1 step 4 has an explicit gate: "Wait for the user to confirm the inventory is correct before proceeding." Phase 3 has an explicit gate: "Do not proceed to Phase 4 until the user confirms the direction." Phase 6 has an explicit gate: "Do not proceed to Phase 7 until the user explicitly approves."

Phase 2 has no gate. Line 61 says "Report specific findings" but does not say to pause for acknowledgment before spawning sub-agents or moving to Phase 3. An agent could complete all of Phase 2 — full codebase exploration plus both sub-agent reviews — and flow directly into proposing an approach, with the user never seeing the Phase 2 output before the approach is already formed.

### 2.2 "Low Severity" Is Undefined

Phase 5 line 215: "Findings above low severity that you chose to reject must be presented to the user for their decision."

The adversarial checklist has four tiers: "This is wrong," "This is overcomplicated," "This is missing," "This is unnecessary." The convention checklist has: "This is wrong," "This violates convention," "This is unnecessary," "This could be simpler." Neither checklist maps these to "low severity." An agent must guess. One agent might treat "This is overcomplicated" as low severity; another might treat everything above "This could be simpler" as high. The filtering rule is unenforceable as written. Define which checklist tiers are low vs not.

### 2.3 No Guidance When User Rejects the Approach in Phase 3

The gate at line 108 says "Do not proceed to Phase 4 until the user confirms the direction." It says nothing about what to do if the user rejects the direction or asks for a different approach. Does the agent return to Phase 2? Propose alternatives inline? The silence leaves the agent to improvise at a critical decision point.

### 2.4 Phase 5 Convention Sub-Agent Gets a Code-Diff Checklist Applied to a Text Plan

Phase 5 tells the convention sub-agent to load `checklists/convention-review.md` and evaluate simplicity, capability ordering, pattern compliance, and completeness. The convention checklist begins: "Run `git diff` (or whatever the calling skill specifies) to see changes." An implementation plan has no diff. The checklist then checks Hotwire Turbo Frame IDs, `authorize` declarations, and `current_merchant` query scoping — none applicable to a text plan.

The sub-agent will either ignore most of the checklist (making the review inconsistent) or apply it nonsensically. Phase 5 needs either a different checklist, a plan-specific review template, or explicit instructions about which checklist sections apply to a design plan vs code.

---

## 3. Missing — Scenarios the Skill Doesn't Handle

### 3.1 Issue Already Has a Partial Plan

The description says "Use after /define or on any issue with a behavior inventory." An issue might already have a partial implementation plan appended from a previous /design run. The skill gives no instruction. Should the agent update the existing plan, replace it, or fail and ask? In Phase 7, `gh issue edit` replaces the issue body — if an agent naively follows "append the implementation plan," it might overwrite the existing plan without preserving the original spec content above it.

### 3.2 Phase 3 Approach Presentation Has No Format Constraints

Phase 3 says "Present it directly. Go deep" with three bullet points. There is no format constraint, no word limit, no required sections, no example. The craft doc (`24-skill-writing-craft.md`) requires explicit output format with constraints for phases that produce user-visible output. Phase 4 is highly structured. Phase 3 is completely freeform — an agent might write three sentences or three pages.

### 3.3 Greenfield Projects With No Existing Similar Feature

Phase 2 step 1: "Find the most similar existing feature and study its model, controller, views, tests, and patterns end to end. This is your template." If the feature is greenfield with no similar prior work, this step yields nothing. The skill gives no fallback. An agent would either invent a template or get stuck.

### 3.4 `gh issue edit` Replaces Body, Not Appends

Phase 7 line 238: "Append the implementation plan to the GitHub issue using `gh issue edit`." The `gh issue edit --body` flag replaces the entire body. To append, the agent must fetch the existing body, concatenate the plan template, and write the combined result. The skill does not mention this. An agent following this literally will lose the original issue content unless it independently reasons through the mechanics of `gh issue edit`.

---

## 4. Convention Violations — Doesn't Follow Craft Conventions

### 4.1 Description Contains Workflow Output Fragment

`00-synthesis.md` and `24-skill-writing-craft.md` state: descriptions must contain only triggering conditions ("Use when..."), never process summaries. Descriptions that summarize the workflow cause Claude to follow the summary and skip the skill body.

The current description contains "needs a technical plan before implementation begins" — this describes what the skill produces, not a trigger condition. It's minor but the criterion is binary: triggering conditions only.

### 4.2 No Explicit Output Format for Phase 3

The craft doc requires an explicit output format with constraints for each user-facing phase output. Phase 4 has a defined format block for capabilities and behavior traceability. Phase 3 has no format specification at all. The plan template covers Phase 4 output but not the Phase 3 approach presentation.

### 4.3 Soft Language in Phase 2 Opening

Line 47: "Understand the terrain before proposing anything." The craft doc requires imperative mood throughout — commands, not descriptions. "Understand" is descriptive framing, not an imperative. Should be "Map the codebase before proposing anything" or similar.

### 4.4 "Ask Questions Freely" Is Vague Non-Imperative Guidance

Rules line 267: "Ask questions freely. Surface uncertainties early." This is the type of vague, generic instruction the craft doc flags as waste — "be thorough"-equivalent. The skill already has specific stop conditions in Failure Modes that tell the agent when to ask. This rule adds noise without changing behavior. Delete it or replace with a specific instruction about what to ask when.

---

## 5. Nitpick — Minor Improvements

### 5.1 Two Sub-Agent Rounds With No Explained Rationale

Phase 2 and Phase 5 both spawn two sub-agents using the same two checklists. Four sub-agent invocations total. The rationale for two rounds (catch direction problems early in Phase 2, audit the full plan in Phase 5) is never stated. An agent or user reading this skill for the first time would find the double round surprising. One sentence explaining the purpose of each round would help.

### 5.2 "Issue Must Stand Alone" Buried in Rules

Line 269: "The issue must stand alone. The updated issue should be clear enough for /execute to implement without the original conversation." This is the most important quality criterion for the final output. It appears only as the last bullet in Rules, far from Phase 7 where it is actionable. It should appear as a checklist item or gate condition in Phase 7.

### 5.3 Context Management Guidance Scoped Too Late

The context management note (lines 243-247) sits inside Phase 7. Context degradation becomes a real risk during the long Phase 4 detail pass, not just at writing time. Moving this note to Phase 4 entry (or making it a standalone top-level note) would place the guidance where it is actionable.

### 5.4 Capability Decomposition Rule May Duplicate CLAUDE.md

Rules line 266: "Behaviors drive capabilities. Never decompose by architectural layer." The craft doc warns against duplicating what CLAUDE.md already enforces — it wastes instruction budget. If this rule is already in CLAUDE.md (which covers capability-ordered decomposition in the commit practices section), it costs slots here without adding behavior. Verify and remove if duplicated.

---

## Summary

| # | Severity | Issue |
|---|---|---|
| 1.1 | Broken | Skill always-produce-alternatives requirement contradicts skill's "clear best approach" shortcut |
| 1.2 | Broken | Checklist paths unresolvable from skill's location |
| 1.3 | Broken | Phase 2 sub-agents get code-diff checklist to assess pre-existing architecture |
| 2.1 | Unclear | Phase 2 has no gate — agent can skip showing exploration to user |
| 2.2 | Unclear | "Low severity" undefined — rejection filtering rule unenforceable |
| 2.3 | Unclear | No path when user rejects Phase 3 approach |
| 2.4 | Unclear | Phase 5 convention sub-agent applies code-review checklist to a text plan |
| 3.1 | Missing | Issues with existing partial plan — no handling |
| 3.2 | Missing | Phase 3 approach presentation has no format constraints |
| 3.3 | Missing | Greenfield projects with no existing similar feature |
| 3.4 | Missing | `gh issue edit` replaces body; "append" instruction is mechanically wrong |
| 4.1 | Convention | Description contains workflow output fragment, not pure trigger |
| 4.2 | Convention | No explicit output format for Phase 3 |
| 4.3 | Convention | "Understand the terrain" is descriptive framing, not imperative |
| 4.4 | Convention | "Ask questions freely" is vague, adds no actionable guidance |
| 5.1 | Nitpick | Double sub-agent round rationale unexplained |
| 5.2 | Nitpick | "Issue must stand alone" buried in Rules, not Phase 7 gate |
| 5.3 | Nitpick | Context management guidance only in Phase 7, should be earlier |
| 5.4 | Nitpick | Capability decomposition rule may duplicate CLAUDE.md |
