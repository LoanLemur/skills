---
name: design
description: >-
  Design the implementation for a defined feature. Use after /define or
  on any issue with a behavior inventory or clear "How it works"
  narrative. Use when the user says "design this", "how should we build
  this", or "plan the implementation". DO NOT use when requirements are
  still unclear or the issue lacks enough detail.
argument-hint: <github-issue-url-or-number>
user-invocable: true
---

# Design

Design the implementation for a feature. Input is a GitHub issue with
clear requirements. Output is an implementation plan — data model,
technical decisions, and an ordered list of capabilities — appended to
the issue.

## Input

$ARGUMENTS

This should be a GitHub issue URL or number. If not provided, ask the
user which issue to work on.

## Phase 1: Understand the Spec

1. Fetch the issue with `gh issue view`. Read it carefully.
2. Identify:
   - **Behaviors** — the behavior inventory (or extract from "How it
     works" if no inventory exists)
   - **Decisions** — product-level decisions already made (respect these)
   - **Constraints** — limitations the implementation must respect
   - **Edge cases** — non-happy-path scenarios to handle
   - **Outstanding questions** — anything unresolved
3. If the issue is missing a behavior inventory, a "How it works"
   narrative, or has unresolved questions that block design: stop. Tell
   the user what's missing and recommend running /define first.
4. Restate the behavior inventory to the user. This is the contract —
   every behavior must have a home in the capability list. Wait for the
   user to confirm the inventory is correct before proceeding.

## Phase 2: Explore the Codebase

Map the codebase before proposing anything. Use Read, Grep, Glob.

1. **The closest existing feature.** Find the most similar feature and
   study its model, controller, views, tests, and patterns end to end.
   This is your template. If no similar feature exists, study 2-3
   existing models/controllers/routes to establish the project's
   conventions.
2. **Conventions** — Enums, concerns, controller structure, test patterns.
   Follow what exists.
3. **Integration points** — How existing code handles the external
   services this feature touches.
4. **Adjacent code** — What this will modify or interact with.
5. **Refactor candidates** — Does existing code need restructuring before
   this feature can be built cleanly? If so, that refactor is the first
   capability.

Report specific findings: list the files, models, and patterns found.
Name what you searched for and where.

### Architectural direction review

Spawn two sub-agent reviews. Provide each with the behavior inventory,
your list of relevant files, and the constraints from the issue.

**Pass 1 — Direction assessment.** Assess whether the behavior inventory
maps to one obvious idiomatic approach or multiple viable approaches.
Flag any codebase constraints that would block the obvious approach.
Flag code smells or structural issues that should be addressed first.

**Pass 2 — Adversarial review.** Load `checklists/adversarial-review.md`
from the skills repository root. Challenge the architectural direction.
Is there a simpler approach? Does any proposed model/table/abstraction
need to exist? What assumptions haven't been verified? What will break?

Spawn sub-agents. Do not perform reviews inline.

Present your codebase findings to the user alongside both assessments.
Highlight where you agree and disagree. If findings change the scope or
reveal unexpected constraints, wait for acknowledgment before proceeding
to Phase 3.

## Phase 3: Propose the Approach

How this phase plays out depends on what Phase 2 revealed.

### When there's a clear best approach

Present it directly. Don't manufacture alternatives — a strawman
comparison wastes time and insults the reader. Go deep:

- The approach and why it's clearly right
- The runner-up approach and why it doesn't work (one sentence minimum
  — this prevents rationalizing a single approach when alternatives
  exist)
- Risks or open questions

### When there are genuinely different options

Present the competing approaches. For each:

- How it works (models, controllers, data flow)
- What it makes easy vs hard
- Concrete tradeoffs (not abstract "flexibility" or "scalability")

Give a clear recommendation.

**Gate: Present the approach (or recommendation among options) to the
user. Do not proceed to Phase 4 until the user confirms the direction.**
If the user rejects the approach, ask what direction they prefer. Return
to Phase 3 with the new constraints.

## Phase 4: Detail the Plan

Only begin this phase after the user confirms the approach from Phase 3.

### Data model

Propose tables, columns, types, and constraints. Null constraints,
defaults, foreign keys, indexes. For each table, justify why it exists.
Always ask: can this be a column on an existing model instead of a new
table?

### Technical decisions

For each significant implementation choice:

- **The decision** as a bold statement
- **Why** — what makes this the simplest correct approach
- **What was considered** — and why it's worse

Always ask: "Can this same value be delivered more simply?"

### Capabilities

List the capabilities to implement, in order. Each capability is the
smallest independently valuable unit — one thing a user or system can
do after it ships that they couldn't before.

Err on the side of too granular, not too grouped. If a capability
contains multiple distinct user actions (create, view, list, edit,
delete), each action that delivers independent value is its own
capability. The executing agent determines commit boundaries per
CLAUDE.md conventions — this list is a starting point, not a contract.

For each capability:

```
N. {Capability name}
   {What this adds, patterns it follows, key nuances.}
```

If you discover behaviors the issue's inventory missed (e.g., technical
behaviors like webhook handling, background jobs), add them with
annotation that they were identified during design.

Propose concrete names for models, controllers, columns, and routes.
Naming reveals design problems early.

### Behavior traceability

Map every behavior from the issue's inventory to one or more capability
numbers:

```
Behavior 1 → Capability N
Behavior 2 → Capabilities N, M
Behavior 3 → Deferred (cross-reference issue)
```

If any behavior has no capability, add one or flag it to the user.

### Implementation notes

Bullet points covering patterns and conventions that apply across
capabilities: patterns to follow, state transitions, calculation
details, conditional validations, cross-references to related issues.

### Testing strategy

Identify which capabilities have non-trivial logic where tests provide
confidence. Default: most code doesn't need tests. Be precise — name
the specific calculation or edge case, not "test the model." The
executing agent makes final testing decisions based on actual code.

## Phase 5: Design Review

Spawn two sub-agent reviews. Provide each with:
- The behavior inventory
- The proposed data model (full schema)
- The proposed capability ordering
- Technical decisions with rationale
- Key existing code patterns being followed (include file paths)

**Pass 1 — Convention review.** Load `checklists/convention-review.md`
from the skills repository root. The convention checklist is designed
for code diffs. When applying it to an implementation plan, evaluate
only: responsibility placement (is logic in the right layer?), DRY (is
anything duplicated across capabilities?), idiom conformance (do
proposed patterns match codebase conventions?). Skip code-specific items
(git diff, Hotwire IDs, tenant scoping). Also evaluate: simplicity (can
anything be eliminated?), capability ordering (does each deliver a real
capability? is the ordering logical?), completeness (every behavior
accounted for?).

**Pass 2 — Adversarial review.** Load `checklists/adversarial-review.md`
from the skills repository root. Challenge: What's overcomplicated?
What assumptions are wrong? What will break during implementation?
What's missing? What contradicts the spec or CLAUDE.md?

Spawn sub-agents. Do not perform reviews inline.

**Act on feedback that:**
- Simplifies the implementation of existing behaviors
- Identifies pattern violations or missed edge cases
- Finds real gaps in the capability list

**Ignore feedback that:**
- Adds behaviors not in the inventory
- Increases complexity without addressing a concrete defect
- Second-guesses product decisions from the issue

Present all review findings to the user. For each finding, state whether
you incorporated it or rejected it and why. Findings categorized as
"This is wrong" or "This is missing" that you chose to reject must be
presented to the user for their decision.

If the review reveals the overall approach is wrong (not just details),
return to Phase 3 rather than patching.

## Phase 6: Finalize with User

Present the complete implementation plan:

- Data model
- Technical decisions (noting any the reviews challenged)
- Capability ordering with behavior mapping
- Implementation notes
- Testing strategy

Verify: could someone implement this feature from the issue alone,
without the conversation?

Do not proceed to Phase 7 until the user explicitly approves. A response
presenting the plan must end without any Phase 7 actions.

## Phase 7: Update the Issue

Once approved, add the implementation plan to the GitHub issue. Use the
template in `assets/plan-template.md` for structure.

Fetch the existing issue body with `gh issue view --json body -q .body`.
If the issue already has an `## Implementation Plan` section, the new
plan replaces it. Otherwise, concatenate the plan below the existing
content with a `---` separator. Write the combined content with
`gh issue edit --body`.

Share the updated issue URL.

### Context management

If this session has been running for 25+ minutes or context feels heavy,
compact before updating the issue. The issue text must be accurate — do
not write from degraded context.

## Failure Modes

- **Spec is incomplete or contradictory:** Stop. State what's missing or
  contradictory. Recommend returning to /define with specific questions.
- **Codebase patterns conflict with spec:** Stop. Present the conflict
  and let the user decide which takes precedence.
- **Design contradicts CLAUDE.md conventions:** Stop. Present the
  contradiction. The user decides.

## Rules

- Never write code. No files created or edited. The only artifact is the
  updated GitHub issue.
- Simplicity first. Always ask "can this be done more simply?"
- Understand the WHY. Read the product spec before proposing. Don't
  re-litigate product decisions already made in the issue.
- Follow existing patterns. The codebase is the style guide.
- Behaviors drive capabilities. Never decompose by architectural layer.
- Be opinionated. Recommend a direction. Defer to the user's judgment.
