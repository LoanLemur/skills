---
name: define
description: >-
  Define a feature or change as a product spec before implementation.
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

## Proportional Ceremony

Not every request needs the full process. Assess scope first:

- **Trivial** (typo, config, one-line fix): Tell the user this doesn't
  need /define. Stop.
- **Small** (1-2 files, clear behavior, single actor): Skip to Phase 3.
  Write a one-paragraph problem statement, 3-5 acceptance criteria, and
  a short behavior inventory. Then Phase 5.
- **Medium+** (3+ files, multiple behaviors or actors, external systems,
  state transitions, edge cases): Full process below.

If the user's description mentions multiple actors, external systems, or
state transitions, it is Medium+ regardless of estimated file count.

State which track you're taking and why. Wait for the user to confirm
the classification before proceeding. If the user disagrees, use their
classification.

## Phase 1: Understand the Problem

Nail down the fundamentals:

1. **What** is being asked for? One sentence.
2. **Why** does this need to exist? What problem does it solve?
3. **Who** is this for?
4. **What does success look like?**
5. **What is explicitly out of scope?**

Present these five answers to the user for confirmation before
proceeding. Ask questions freely throughout — ambiguity resolved now
saves redesign later.

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

Walk through the feature from each actor's perspective. Bold headers
for each phase. Be specific about what the user does, sees, and what
happens next.

At each step: feedback on success, failure, and in-progress states.
Notifications sent, and to whom.

Be opinionated. Recommend a direction. Defer to the user's judgment
when they disagree.

**Length guidance:** Under 500 words for small features, under 1000
words for medium+. If you need more, the scope is likely too large.

### Key decisions

For each significant product choice:

- **The decision** as a bold statement
- **Why** this is the right choice
- **What was considered and rejected** — and why

Product-level decisions only. When a decision has both product and
technical dimensions, include enough context that /design can make the
right technical call.

### Constraints

Anything that constrains implementation:

- External API limitations or behaviors
- Non-obvious business rules
- Things tried before that failed (and why)
- Dependencies on other features or systems

**Gate: Present the solution narrative and key decisions to the user.
Do not proceed to Phase 4 until the user confirms the direction.**

## Phase 4: Edge Cases

Non-happy-path scenarios:

- What triggers it
- What the user sees
- What the system should do

Focus on edge cases that affect product behavior. Implementation edge
cases (null handling, retries) are /design's concern.

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
inventory). Load the shared review checklists from the parent
`checklists/` directory.

**Pass 1 — Spec review.** Load `checklists/spec-review.md`.
Evaluate: Is the behavior inventory complete? Are key decisions
well-reasoned? Are constraints specific enough? Is there enough
information to design an implementation?

**Pass 2 — Adversarial review.** Load `checklists/adversarial-review.md`.
Challenge: Is the scope too large? What assumptions are unstated? What
edge cases are missing? What happens if a key decision is wrong? Is
anything being built that nobody asked for?

Present all review findings to the user. For each finding, state whether
you incorporated it or rejected it and why. Findings rated above low
severity that you chose to reject must be explicitly presented to the
user for their decision — do not reject substantive findings unilaterally.

Spawn sub-agents for reviews. Do not perform reviews inline in the
current context — the point is independent evaluation.

## Phase 7: Create GitHub Issue

### Completion check

Before presenting to the user, re-read the original request ($ARGUMENTS).
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
- Capture what was rejected. Rejected alternatives and constraints are
  as valuable as the chosen direction.
- Ask questions freely. Ambiguity resolved now saves redesign later.
- Be opinionated. Recommend a direction. Defer to the user's judgment.
- The issue must stand alone. Someone running /design on this issue
  should have everything they need without the original conversation.
