# Scenario: /define — Balance Calculation Feature

## User Prompt

> I want to add balance calculation to Splitwise Lite. Users should be
> able to see who owes whom in a group, and get suggestions for how to
> settle up with minimal transactions.

## Context: Existing Codebase

Splitwise Lite is an Express/Node/SQLite expense-splitting API. What
exists today:

- **Groups**: CRUD endpoints (`/groups`). A group has a name and
  created_at.
- **Members**: CRUD endpoints (`/groups/:groupId/members`). A member
  belongs to a group and has a name and email.
- **Expenses**: CRUD endpoints (`/groups/:groupId/expenses`). An expense
  has an amount, description, payer_id (references a member), and
  created_at. Expenses are split equally among all group members.

What does NOT exist:

- No balance calculation (who owes whom)
- No settlement suggestions (minimal transactions to settle debts)
- No payment/settlement recording
- No non-equal split types (percentage, exact amounts)
- No multi-currency support

## Grading Rubric

An evaluator agent scores each item as PASS / PARTIAL / FAIL.

### 1. Problem Statement

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| States what is broken or missing today | Names the specific gap: expenses exist but there is no way to see net balances or settle debts | Mentions the gap but vaguely ("needs improvement") | No problem statement, or jumps straight to solution |
| Names the actors | Identifies group members as the primary actors | Mentions "users" generically | No actors identified |
| States the goal | One clear sentence: what the system can do after this ships | Goal is stated but ambiguous or compound | No goal |

### 2. Behavior Inventory

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Numbered list of discrete behaviors | Each behavior has a trigger, state change, and outcome | Behaviors listed but some are vague ("handle balances") | No inventory, or behaviors are paragraphs not a list |
| Covers balance viewing | A behavior for retrieving per-member balances within a group | Mentioned but not as a distinct behavior | Missing |
| Covers pairwise debts | A behavior for seeing who owes whom (not just net balance) | Partially described | Missing |
| Covers settlement suggestion | A behavior for computing minimal transactions to settle | Mentioned but bundled with another behavior | Missing |
| Covers settlement recording | A behavior for recording that a settlement occurred, or explicitly defers it with rationale | Either covered or explicitly deferred | Not mentioned at all |
| Behaviors are decomposed | No behavior contains vague verbs like "manage", "handle", "process" without specifics | Most are specific but 1-2 are vague | Multiple vague behaviors |

### 3. Edge Cases

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Empty group (no members or no expenses) | Explicitly addressed: what does the balance endpoint return? | Mentioned but no expected behavior stated | Not mentioned |
| Zero-amount expense | Addressed: is it allowed? What happens to balances? | Mentioned in passing | Not mentioned |
| Rounding errors | Addressed: how are fractional cents handled when splitting? (e.g., $10 split 3 ways) | Acknowledged but no resolution | Not mentioned |
| Single-member group | Addressed: payer is the only member | Mentioned | Not mentioned |
| Member added after expenses exist | Addressed: do old expenses retroactively include new member? | Mentioned | Not mentioned |
| Self-payment in pairwise view | Addressed: payer's debt to themselves is excluded or zero | Mentioned | Not mentioned |

### 4. Constraints

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Currency scope | Explicitly states single-currency (or states multi-currency is out of scope) | Implied but not explicit | Not mentioned |
| Split type scope | Explicitly states equal-split only (or defines which split types are in scope) | Implied | Not mentioned |
| Settlement scope | States whether partial settlements are allowed or if settlements must be exact | Mentioned | Not mentioned |

### 5. Key Decisions

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Split type decision | Explicit decision: equal-split only for v1 (or whatever the decision is), with rationale | Decision made but rationale missing | No decision — left ambiguous |
| Settlement algorithm | Decision on approach: e.g., debt simplification to minimize transactions, or simple pairwise | Mentioned but not decided | Not mentioned |
| Balance calculation timing | Decision: computed on-read vs stored/materialized | Mentioned | Not addressed |

### 6. Spec Completeness

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Handoff-ready for /design | A developer reading only this issue could produce an implementation plan without asking clarifying questions | 1-2 minor ambiguities remain | Major gaps — /design would need to guess or ask multiple questions |
| Free of implementation details | No mention of specific tables, columns, algorithms, or file paths — purely WHAT, not HOW | Minimal leakage (e.g., mentions "endpoint" which is borderline) | Specifies data model, code structure, or specific algorithms |
| Rejected alternatives captured | At least one rejected approach or scope reduction is documented | Alternatives mentioned but reasons not given | No alternatives documented |
