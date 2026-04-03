# Evaluation: /define Skill

Evaluated against: `24-skill-writing-craft.md`, `28-skill-map.md`, `00-synthesis.md`

---

## 1. Broken — Agent would produce wrong output or get stuck

**B1: Small track skips Phase 6 (review) — sub-agents never spawn for small features.**

The Proportional Ceremony section (lines 34-38) says the small track goes "Skip to Phase 3 ... Then Phase 5." Phase 6 (sub-agent reviews) is entirely omitted. An agent following the small track will produce an issue that has never been adversarially reviewed. This is intentional-looking but the skill never states whether Phase 6 is optional for small features. An agent reading it literally will either skip reviews (producing lower-quality output) or get confused about what "Then Phase 5" means — does it mean stop at Phase 5, or continue through 6 and 7?

**Fix:** Explicitly state whether review is required for small features. If not required, say so. If a lighter version applies (e.g., single-pass spec review only), specify that.

**B2: Phase 6 references `checklists/` with a relative path that is ambiguous.**

Lines 191-196 say: "Load the shared review checklists from the parent `checklists/` directory." and "Load `checklists/spec-review.md`." The `define/` skill directory has no `checklists/` subdirectory. The actual checklists live at `skills/checklists/`, one level above `skills/define/`. "Parent `checklists/` directory" is ambiguous — parent of what? Of `define/`? Of `SKILL.md`? An agent spawning a sub-agent needs a file path, not a relative hint. Without an absolute or repo-root-relative path, the sub-agent may fail to find the file and either hallucinate the checklist content or error out silently.

**Fix:** Use an explicit path from the repo root, e.g., `skills/checklists/spec-review.md`, or document the path resolution convention once at the top of the skill.

**B3: "Then Phase 5" in the small track is wrong — Phase 4 is skipped without notice.**

The small track says: "Skip to Phase 3. Write a one-paragraph problem statement, 3-5 acceptance criteria, and a short behavior inventory. Then Phase 5." Phase 4 (Edge Cases) is silently dropped. A behavior inventory without edge cases may be incomplete for small features that have non-obvious failure paths. More critically, the instruction to write "3-5 acceptance criteria" in Phase 3 has no equivalent in the full process — acceptance criteria are not mentioned anywhere in Phases 3-5 for medium+ features. This creates two inconsistent output structures for the same issue template.

**Fix:** Either explicitly say "Phase 4 is skipped for small features" or include a lightweight edge case step. Reconcile what "acceptance criteria" means vs. the behavior inventory — they may be the same thing, but the skill uses both terms in different places without defining the relationship.

---

## 2. Unclear — Agent might interpret multiple ways, some wrong

**U1: Phase gate after Phase 1 is implicit; gate after Phase 2 is absent.**

Phase 3 has an explicit gate: "Present the solution narrative and key decisions to the user. Do not proceed to Phase 4 until the user confirms the direction." (line 150-151). Phase 1 says "Present these five answers to the user for confirmation before proceeding" (line 57) — that's a gate. Phase 2 has no gate at all. An agent that completes Phase 2 reconnaissance may immediately proceed to Phase 3 without surfacing findings to the user. The user may want to redirect scope based on what exists in the codebase.

**Fix:** Add an explicit gate after Phase 2: present findings and confirm they don't change the scope before proceeding.

**U2: "Ask questions freely throughout" conflicts with the instruction to present findings as recommendations.**

Lines 57-59 say "Ask questions freely throughout — ambiguity resolved now saves redesign later." Lines 85-87 say "State findings as recommendations, not questions. 'X should ship separately because Y' — not 'have you considered shipping X separately?'" These two directives are in direct tension. An agent may not know when to question vs. when to assert. The "challenge the premise" section resolves this for that specific activity, but "ask questions freely" is a global rule that contradicts it.

**Fix:** Replace "Ask questions freely throughout" with something specific: clarifying questions about missing facts are fine; product/scope conclusions should be stated as recommendations.

**U3: The "2-3 pointed questions" instruction in Phase 1 is ambiguous about timing.**

Lines 92-93: "Then ask 2-3 pointed questions — the specific uncomfortable questions this particular feature raises. Wait for answers before proceeding." It's unclear whether the agent should ask these questions after presenting all findings from the challenge-the-premise scan, or interleaved. It's also unclear whether these questions replace or supplement the Phase 1 confirmation (lines 57-59). A literal reading suggests the agent presents the 5 answers, then presents challenge findings, then asks 2-3 questions — three separate user interactions before leaving Phase 1. That may be excessive for a small feature. But for medium+ features there's no gate separating the 5-answer confirmation from the challenge findings.

**Fix:** Consolidate: present the 5-answer summary and the challenge recommendations together, then ask the 2-3 questions. One user interaction per phase, not three.

**U4: "Phase 5 — Behavior Inventory" is ambiguous about whether it replaces or supplements acceptance criteria.**

The small track instructs the agent to write "3-5 acceptance criteria" as part of the Phase 3 output. The full track produces a behavior inventory in Phase 5. The issue template has only a "Behavior Inventory" section — no "acceptance criteria" section. A fresh agent reading the small track will write acceptance criteria with no place to put them in the issue template.

**Fix:** Remove "acceptance criteria" from the small track description, or define how they map to the behavior inventory.

**U5: The completion check in Phase 7 says to re-read "$ARGUMENTS" but this may not be available.**

Line 213: "Before presenting to the user, re-read the original request ($ARGUMENTS)." If the user invoked the skill through a slash command with typed arguments, `$ARGUMENTS` will be populated. But if the feature emerged from multi-turn conversation (the user said "I want to add X" in prior messages), `$ARGUMENTS` may be empty or contain only what was passed at invocation. The skill doesn't tell the agent what to do when `$ARGUMENTS` is empty — check the conversation history? Ask the user to restate? This is a real failure path.

**Fix:** Specify the fallback: "If $ARGUMENTS is empty, use the feature description from the conversation as the original request."

---

## 3. Missing — Scenarios the skill doesn't handle

**M1: No guidance on what to do when the user disagrees with a Phase 1 recommendation.**

The skill says "Be opinionated. Recommend a direction. Defer to the user's judgment when they disagree." (line 252-253). But there's no instruction on what deferring actually means. Does the agent: (a) note the disagreement in the issue and proceed with the user's direction, (b) present both options in the issue for /design to decide, or (c) silently adopt the user's position? For product decisions that affect scope or viability, the rejected recommendation may be valuable for /design to see as a constraint. The skill says "Capture what was rejected" (line 249) but doesn't apply this to cases where the agent's own recommendation is rejected.

**Fix:** When the user overrides a recommendation, the rejected direction and the user's reasoning should appear in the issue's "Key decisions" section.

**M2: No failure mode for when sub-agent review fails or returns no findings.**

Phase 6 specifies what to do with findings but not what to do if the sub-agent can't be spawned, returns an error, or returns "no issues found." The agent may silently proceed, or get stuck waiting. "No issues found" from an adversarial review is also suspicious and may indicate the review was shallow.

**Fix:** Add: "If a review sub-agent returns no findings, note this in your summary to the user. Do not treat absence of findings as confirmation the spec is complete."

**M3: No guidance on how to handle a user who says "just write the issue" mid-process.**

Users frequently truncate process skills. If a user says "skip all this, just write the issue" after Phase 1, the skill has no instruction for this. The agent will likely comply (it's trained to be helpful) and produce an issue that bypasses Phases 2-6, potentially missing codebase constraints (Phase 2) and adversarial review (Phase 6).

**Fix:** Add an explicit rule: "If the user asks to skip phases, complete at minimum: Phase 2 (codebase scan) and Phase 6 (review). These cannot be skipped because Phase 2 reveals constraints that change scope, and Phase 6 catches issues a single-pass definition misses. Explain this to the user."

**M4: No instruction on issue labeling, milestone, or project assignment.**

The skill says create the issue with `gh issue create` using HEREDOC, then share the URL. GitHub issues often need labels (e.g., `feature`, `enhancement`) and project assignments. The issue template has no title instruction. The `gh issue create` command without `--label` or `--project` flags produces an unlabeled issue. This may be intentional (let the user handle it) but the skill doesn't say so.

**Fix:** Either specify that labels/projects are out of scope for /define, or add a step: "Apply the `feature` label and assign to the active milestone if one exists."

---

## 4. Convention Violations — Doesn't follow craft conventions but wouldn't cause agent failure

**CV1: Description field contains a workflow summary in the first sentence.**

Line 3-4: "Define a feature or change as a product spec before implementation." Per `00-synthesis.md` and `24-skill-writing-craft.md`, the description must contain only triggering conditions ("Use when..."), never process summaries. This first sentence summarizes what the skill does, not when to use it. An agent reading this description to decide whether to activate the skill may follow the summary and skip the skill body.

**Fix:** Start with a triggering condition: "Use when the user describes a new feature idea..." (which is the second sentence). Delete or rewrite the first sentence.

**CV2: The description word count is at the boundary.**

The description is approximately 80 words, within the 50-150 word requirement. But it's worth noting that the first sentence is waste by convention — removing it brings the count to ~65 words, which is better.

**CV3: Phase headers use H2 but sub-sections use H3 — inconsistent with the issue template which uses H2 for all sections.**

The issue template (`assets/issue-template.md`) uses H2 for Problem, Goal, Design, Behavior Inventory, and Outstanding Questions. The SKILL.md uses H2 for phases and H3 for sub-sections within phases (Key decisions, Constraints). This isn't a problem for the skill itself, but "Key decisions" and "Constraints" in Phase 3 map to sections in the issue template. A fresh agent may use H3 in the issue body, inconsistent with the template's H2 convention.

**CV4: No example output is provided anywhere.**

The craft guide (`24-skill-writing-craft.md`, lines 56-75) is explicit: "One real code snippet outperforms three paragraphs of description." The skill has no example of what a good behavior inventory entry looks like, no example of a well-formed key decision, no example of a vague vs. specific constraint. These are the most likely places an agent will produce low-quality output, and the absence of examples is a missed opportunity.

**Fix:** Add 2-3 concrete examples inline or in `references/`. At minimum: one example behavior inventory entry (trigger, state change, outcome) and one example key decision (chosen, why, rejected alternative, why rejected).

**CV5: "Ask questions freely" appears in both Phase 1 and the Rules section — duplicated.**

Line 59 and line 252 both say essentially the same thing. Duplication inflates the skill's token cost and creates the risk that an agent treats them as two separate (possibly conflicting) instructions.

**Fix:** State it once, in Rules.

---

## 5. Nitpicks — Minor improvement opportunities

**N1: "Nail down the fundamentals" (line 49) is informal and not imperative.**

Should be: "Answer these five questions." The skill otherwise uses imperative voice well; this is an outlier.

**N2: "This is reconnaissance, not design" (line 111) is a description, not an instruction.**

Could be cut entirely without changing behavior, since the next sentence ("Understand what's possible and constrained") is the actual instruction. Per the craft guide, every line must justify its existence by changing behavior.

**N3: The Rules section at the bottom duplicates instructions that appear inline.**

"Ask questions freely," "Be opinionated," and "Capture what was rejected" all appear earlier in the skill. The Rules section reads as a summary of things already said. The craft guide warns against over-specification. If a rule is in the right place inline, it doesn't also need to be in a Rules section.

**N4: Phase numbering creates a false gap in the small track.**

The small track says "Skip to Phase 3 ... Then Phase 5." Skipping Phase 2 means the agent skips codebase exploration entirely for small features. This may be intentional (small, single-file changes don't need reconnaissance) but skipping Phase 2 entirely seems wrong for any change that touches existing code. A new endpoint on an existing model is "small" (1-2 files) but codebase exploration would reveal adjacent patterns to follow.

**N5: The "Outstanding Questions" section in the issue template is not mentioned in any phase.**

The template has an "Outstanding Questions" section but no phase in the skill produces content for it. Phase 6 may surface questions that weren't resolved, but the skill tells the agent to present findings and incorporate or reject them — not to carry unresolved questions into the issue. The template field is either vestigial or the skill needs a step that populates it.

**Fix:** Either remove the section from the template or add: "Questions that remain open after review go into the Outstanding Questions section."

---

## Static Check Summary

| Check | Result |
|---|---|
| Description 50-150 words | Pass (~80 words) |
| Description contains only triggering conditions | Fail — first sentence is a workflow summary |
| SKILL.md under 500 lines | Pass (255 lines) |
| YAML frontmatter well-formed | Pass |
| No sycophantic language | Pass |
| Explicit output format with constraints | Pass (issue template + word limits) |
| Explicit "stop and report" failure modes | Partial — three failure modes listed but "cannot spawn sub-agent" is missing |
| No persona framing | Pass |
| Imperative voice throughout | Mostly pass — two informal phrases noted |
| Specific and literal (not vague) | Mostly pass — "ask questions freely" is vague |
| Phase gates clear | Partial — Phase 2 has no gate; small track phase transitions are ambiguous |
| Output template matches /design's input needs | Pass — issue template maps to what /design expects |

---

## Goal-Level Check (against skill-map.md)

The skill-map says /define must produce: "GitHub issue with problem, goal, behaviors, decisions, constraints, edge cases, behavior inventory." The issue template covers all of these. The skill-map also notes: "explicit 'stop and report' when input is too vague." This is present. The skill-map's acceptance criteria require passing adversarial testing — B1 (small track skips reviews) would fail this test if an adversarial agent took the small track path.

The skill-map entry for /define is silent on whether Phase 6 reviews are required for small features. This ambiguity in the skill-map propagates into the skill. The skill-map should be updated to specify whether lightweight features need review or not, and the skill should reflect that decision explicitly.
