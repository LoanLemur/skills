# Skill Map v2

After red team review (devil agent) and practitioner input.

---

## The Skills (5)

```
/forge (built first, by hand)
  │
  ├── builds & tests ──→ /define
  ├── builds & tests ──→ /design
  ├── builds & tests ──→ /execute
  └── builds & tests ──→ /review
```

### Pipeline

```
User request
     │
     ▼
  Human decides route:
     │
     ├──→ trivial ──→ just do it (no skill)
     ├──→ small/clear ──→ /execute directly
     ├──→ needs exploration ──→ spike manually, then /define
     └──→ known requirements ──→ /define
                                    │
                                    ▼
                                 /design
                                    │
                                    ▼
                                 /execute
                                    │
                                    ▼
                                 /review (includes spec cross-reference)
                                    │
                                    ▼
                                 PR / merge
```

Pipeline routing is a human decision, not agent self-orchestration.

---

### 1. /forge
**Job:** Build, test, and iterate on skills using a Ralph-style loop
**Input:** Skill spec (from synthesis + this map) + acceptance criteria + existing skill (if enhancing)
**Output:** A tested skill that passes adversarial, two-Claude, and comparison tests
**Built:** First, by hand (there is no forge to forge the forge)
**Key mechanism:** Ralph loop — iterate with fresh context until acceptance criteria are met. Only surfaces to the user when it passes all tests or gets stuck.

### 2. /define
**Job:** Define WHAT to build and WHY as a product spec
**Input:** User's requirements
**Output:** GitHub issue with problem, goal, behaviors, decisions, constraints, edge cases, behavior inventory
**Built by:** /forge (enhanced from existing skill)
**Key enhancements from research:** Anti-sycophancy, proportional ceremony (trivial tasks get a lightweight issue), completion-bias countermeasures, explicit "stop and report" when input is too vague

### 3. /design
**Job:** Design HOW to build it as a technical plan
**Input:** GitHub issue from /define
**Output:** Implementation plan appended to the same issue
**Built by:** /forge (enhanced from existing skill)
**Key enhancements from research:** Must structurally produce competing approaches before selecting one (anti-myopia, not advisory), context management instructions, explicit "stop and report" when the spec is discovered to be wrong or incomplete

### 4. /execute
**Job:** Implement the plan commit by commit with verification
**Input:** GitHub issue with implementation plan from /design
**Output:** Commits on a feature branch, deviations as issue comments
**Built by:** /forge (enhanced from existing skill)
**Key enhancements from research:** Context management (compact every 3-4 commits), test-first/test-after mode selection, explicit "stop and report" when the design won't work

### 5. /review
**Job:** Code review with history surgery + goal verification
**Input:** Completed feature branch + original issue
**Output:** Clean commit history, verification report (behavior inventory cross-referenced against commits)
**Built by:** /forge (enhanced from existing skill)
**Key enhancements from research:** Merge /finish's spec cross-referencing into this skill, rule-compliance checklist derived from CLAUDE.md, selective context transfer (spec + checklist, not conversation history)

---

## Review Mechanism

Based on research (doc 29): persona framing ("You are DHH") provides no benefit and may slightly hurt coding task accuracy (PRISM paper, March 2026: -0.65 decline). What actually works is the checklists, context isolation, and posture instructions.

**Replace persona-based agents with checklist-based sub-agents:**
- `convention-reviewer` — Convention compliance checklist (was "DHH"). Can run on Haiku for ~80% cost reduction.
- `adversarial-reviewer` — Adversarial challenge framework (was "Devil"). Can run on Haiku.
- Checklists live in `references/convention-review-checklist.md` and `references/adversarial-review-checklist.md` — shared across all skills that need review.
- No `.claude/agents/` persona files needed. Skills spawn plain sub-agents with the checklist as context.

**Two passes, not three.** Convention + adversarial is sufficient. The third pass (second convention) can be dropped — the evidence doesn't support the marginal value over the cost.

---

## What's NOT a Skill (But Still Exists)

**Existing skills kept as-is:** /qa, /pr-review, /retro — already working, not part of this build. Enhance independently later if needed.

**Cross-cutting concerns:** Shared `references/cross-cutting.md` loaded by every skill. Contains anti-sycophancy instructions, context management guidance, output format constraints. One file, one place to update.

**Review checklists:** Shared `references/convention-review-checklist.md` and `references/adversarial-review-checklist.md`. Loaded by any skill that needs review. One place to update.

**Output verbosity enforcement:** A PostToolUse or Stop hook (deterministic, ~20 lines) that flags output exceeding a threshold. Built before any skill.

**Pipeline routing:** Human decision. Not a skill.

**Spike/exploration:** Manual for now. If it proves needed frequently, build as a skill later.

---

## Build Order

1. **Output verbosity hook** — Deterministic enforcement, applies to everything. ~20 lines.
2. **`references/cross-cutting.md`** — Shared cross-cutting concerns. One file.
3. **/forge** — Built by hand. The skill that builds and tests all other skills.
4. **/define** — First skill built by /forge. Enhanced from existing.
5. **/design** — Enhanced from existing.
6. **/execute** — Enhanced from existing.
7. **/review** — Enhanced from existing, with /finish merged in.

---

## Acceptance Criteria (for /forge to test against)

Every skill must:
- Pass adversarial testing (agent can't find loopholes or skip steps)
- Pass two-Claude evaluation (separate Claude evaluates output against goal, not task)
- Produce output that is not verbose (under defined thresholds)
- Handle its "stop and report" failure mode (wrong input, discovered problems)
- Not contain sycophantic language in any output
- Follow the Agent Skills standard (SKILL.md format, progressive disclosure)
- Outperform or match the existing version on a comparison test (for enhanced skills)

---

## Failure Modes (every skill transition)

Each skill has an explicit "stop and report" output for when its input is wrong:
- **/define:** "This request is too vague to spec. Here's what I need clarified: [specific questions]"
- **/design:** "The spec is incomplete or contradictory. Here's what's wrong: [specific issues]. Return to /define."
- **/execute:** "The design won't work in practice. Here's why: [specific problems]. Return to /design."
- **/review:** "The implementation doesn't match the spec. Here's what's missing: [behavior inventory gaps]."

Backward loops are human decisions, not automated. The skill makes the failure visible; the human decides what to do.
