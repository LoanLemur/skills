---
name: define
description: >-
  Use when the user describes a new feature idea, requests new
  functionality, or needs to scope a change that touches multiple files
  or actors. Use when someone says "I want to add...", "we need...", or
  "let's build...". DO NOT use for bug fixes with obvious scope, config
  changes, typo fixes, or work where requirements are already in an issue.
argument-hint: <feature or change description>
user-invocable: true
---

# Define

Define a feature through structured product discussion. Output is a
GitHub issue specifying WHAT and WHY — clear enough that /design can
produce an implementation plan from the issue alone.

## Input

$ARGUMENTS

If the input is too vague to scope ("make it better", "improve
performance"), stop. State what's missing and ask specific questions.
Do not guess at requirements.

## Output discipline

The issue is for humans who scan, not readers who parse prose. Every
section inside the issue obeys these rules:

- Structured bullets, not paragraphs. One takeaway per bullet.
- Short sentences. Cut filler words.
- Blank line between findings for parseability.
- Omit empty sections entirely. No "(none)" filler, no placeholder
  headers.
- No preamble ("Here's the design..."), no wrap-up ("In summary...").
- Numbered lists only where sequence or the Behavior Inventory
  contract requires.

Self-check before presenting: can a reader scan the whole issue in
under 60 seconds? If not, cut more.

## Proportional Ceremony

Not every request needs the full process. Assess scope first:

- **Trivial** (typo, config, one-line fix): Tell the user this doesn't
  need /define. Stop.
- **Small** (1-2 files, clear behavior, single actor): Skip to Phase 3.
  Write a one-paragraph problem statement and a short behavior inventory
  (3-5 items). Then Phase 5, then Phase 7. Skip Phase 6 — small changes
  don't justify sub-agent review overhead.
- **Medium+** (3+ files, multiple behaviors or actors, external systems,
  state transitions, edge cases): Full process below.

If the user's description mentions multiple actors, external systems, or
state transitions, it is Medium+ regardless of estimated file count.

State which track you're taking and why. Wait for the user to confirm
the classification before proceeding. If the user disagrees, use their
classification.

## Phase 1: Understand the Problem

Answer these five questions:

1. **What** is being asked for? One sentence.
2. **Why** does this need to exist? What problem does it solve?
3. **Who** is this for?
4. **What does success look like?**
5. **What is explicitly out of scope?**

Present these five answers to the user for confirmation before
proceeding. Ask clarifying questions about missing facts. State product
conclusions as recommendations.

### Challenge the premise

Stress-test whether the right thing is being built. Scan these areas
for gaps or unstated assumptions:

- **Scope** — Is this the smallest version that delivers value? What
  could ship later? What's being added "while we're in here" that isn't
  core?
- **Domain model** — Are entities and relationships clear? Ambiguous
  ownership or lifecycle? Can this be a column on an existing model
  instead of a new table?
- **User interactions** — Is every actor's flow specific? Any vague
  verbs ("manage", "handle") hiding undefined behavior? What does the
  user see on failure?
- **Integration points** — What external systems are involved? Failure
  modes? Timeouts? Rate limits?
- **Edge cases** — What happens at boundaries? Empty states, concurrent
  access, partial failures?
- **Constraints** — Regulatory, compliance, financial implications?
  Known limitations from past attempts?
- **Business impact** — What changes if this succeeds? Who else is
  affected that hasn't been named?
- **Assumptions** — What's taken for granted? What happens if wrong?

State findings as recommendations, not questions. "X should ship
separately because Y" — not "have you considered shipping X separately?"

You must surface at least one recommendation for scope reduction and one
unstated assumption. If you genuinely cannot, explain why this feature
is unusually well-defined.

Then ask 2-3 pointed questions — the specific uncomfortable questions
this particular feature raises. Wait for answers before proceeding.

## Phase 2: Understand What Exists

Explore the current system. Use Read, Grep, Glob — enough to understand
the current user experience and adjacent features.

1. **Current experience** — How does the user handle this today?
2. **Adjacent features** — What existing features relate? What patterns
   do they follow?
3. **Integration points** — What external services does this touch?
4. **Constraints** — Rate limits, API restrictions, third-party
   limitations, business rules not obvious from the description.

Report specific findings: list the files, models, or patterns found.
If you explored and found nothing adjacent, name what you searched for
and where.

This is reconnaissance, not design. Understand what's possible and
constrained.

## Phase 3: Define the Solution

Write the solution as structured bullets, not narrative. Walk through
the feature from each actor's perspective:

- One bold header per actor (e.g. `**MERCHANT**`, `**CUSTOMER**`,
  `**SYSTEM**`).
- Under each header, 3-6 short bullets. Each bullet names an action
  and its outcome. Include feedback on success, failure, and
  in-progress states, plus notifications sent and to whom.
- Blank line between actor blocks.

Be opinionated. Recommend a direction. Defer to the user's judgment
when they disagree.

**Length caps:** Under 200 words for small features, under 400 words
for medium+. If you need more, the scope is likely too large —
recommend splitting.

### Key decisions

One bullet per product decision. Format:

- Decision in one short sentence. Rationale in one short sentence.
  Indented follow-up line names rejected alternatives with a one-phrase
  reason each.

Example:
- Stripe Checkout, not custom. PCI scope small, 3DS free.
  Rejected Elements (still PCI), custom (compliance cost).

Blank line between decisions. Product-level decisions only. When a
decision has both product and technical dimensions, include enough
context that /design can make the right technical call.

### Constraints

One bullet each. Anything that constrains implementation:

- External API limitations or behaviors
- Non-obvious business rules
- Things tried before that failed (and why)
- Dependencies on other features or systems

At minimum, address these scope questions in the bullets:

- Units/currencies/formats in scope vs out of scope
- Operations atomic vs allow partial completion
- Volume/scale assumptions bounding the design

Omit the section entirely if nothing applies.

**Gate: Present the solution bullets and key decisions to the user.
Do not proceed to Phase 4 until the user confirms the direction.**

## Phase 4: Edge Cases

One bullet per edge case. Format: `trigger → outcome`. Keep each to
one line where possible. Blank line between bullets.

Example:
- Zero amount → form blocks submit, "Amount must be > 0". No record.
- SMS delivery fails → request stays Pending, merchant sees delivery error.

Scan these categories for edge cases specific to this feature:

- **Zero/null inputs** — Zero amounts, empty strings, missing fields.
- **Temporal ordering** — Entities added or removed after related data
  exists.
- **Self-referential states** — Same entity on both sides of a
  relationship.
- **Boundary values** — Exactly 0, 1, or MAX.
- **Concurrent/partial states** — Operation interrupted halfway.

Focus on edge cases that affect product behavior. Implementation edge
cases (null handling, retries) are /design's concern. Omit the section
entirely if no product-visible edge cases exist.

## Phase 5: Behavior Inventory

Decompose the solution into a flat numbered list of discrete
capabilities. A behavior is distinct if it has its own user interaction,
state transition, or integration point.

Each behavior must name a specific trigger, a specific state change, and
a specific outcome. If a behavior contains verbs like "manage", "handle",
or "process" without specifying what that means, it is not yet
decomposed.

Verify:

- Every behavior from Phase 3 is represented
- Each behavior is distinct (not a sub-step of another)
- Behaviors deferred to other issues are called out

This inventory is the contract between /define and /design. Every
behavior must eventually have a commit.

## Phase 6: Review

Spawn two sub-agent reviews. Provide each with the complete draft
(problem, goal, solution, decisions, constraints, edge cases, behavior
inventory).

**Pass 1 — Spec review.** Load the `checklists/spec-review.md` file
from the skills repository root. Evaluate: Is the behavior inventory
complete? Are key decisions well-reasoned? Are constraints specific
enough? Is there enough information to design an implementation?

**Pass 2 — Adversarial review.** Load the `checklists/adversarial-review.md`
file from the skills repository root. Challenge: Is the scope too large?
What assumptions are unstated? What edge cases are missing? What happens
if a key decision is wrong? Is anything being built that nobody asked for?

Present all review findings to the user. For each finding, state whether
you incorporated it or rejected it and why. Findings rated above low
severity that you chose to reject must be explicitly presented to the
user for their decision — do not reject substantive findings unilaterally.

Unresolved questions from reviews go into the Outstanding Questions
section of the issue.

Spawn sub-agents for reviews. Do not perform reviews inline in the
current context — the point is independent evaluation.

## Phase 7: Create GitHub Issue

### Completion check

Before presenting to the user, re-read the original request ($ARGUMENTS).
If $ARGUMENTS is empty, use the feature description from the conversation.
Present a visible mapping:

```
Original request → Issue coverage:
- [request item 1] → [section where addressed]
- [request item 2] → [section where addressed]
- [request item 3] → deferred because [reason]
```

If anything from the original request is not addressed, add it or
explicitly note why it was deferred.

### Create the issue

Load `assets/issue-template.md` and use it as the structure for the
issue body. Present the full issue to the user for review. Once approved, create with
`gh issue create` using HEREDOC for the body.

After creating, share the URL.

## Failure Modes

- **Input too vague:** Stop. State what's missing. Ask specific
  questions. Do not guess.
- **Scope too large:** The feature cannot be captured in a single issue
  with a behavior inventory under ~15 items. Recommend splitting.
  Identify the smallest version that delivers value. Ask the user which
  piece to define first.
- **Cannot explore codebase:** State what you need access to and stop.

## Rules

- Never write code. The only artifact is a GitHub issue.
- No implementation plans. No data models, column types, commit plans,
  or file lists. That's /design's job.
- Codebase Context in the issue records patterns and constraints, not
  file paths or implementation details. Write "uses a nested REST
  pattern" not "see src/routes/expenses.js".
- Capture what was rejected. Rejected alternatives and constraints are
  as valuable as the chosen direction.
- If the user asks to skip phases, Phase 2 (codebase scan) cannot be
  skipped — it reveals constraints that change scope.
- The issue must stand alone. Someone running /design on this issue
  should have everything they need without the original conversation.
