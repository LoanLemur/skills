# Scenario: /design — Balance Calculation Feature

## Input: Completed /define Spec

The following is a realistic product spec (what /define should produce).
Provide this as the GitHub issue body when testing /design.

---

## Problem

Splitwise Lite tracks group expenses but has no way to answer the core
question: who owes whom? Members can record expenses but cannot see
their net balance within a group or get guidance on how to settle debts
efficiently. Without this, the app is a ledger with no payoff.

## Goal

Members can view per-member balances within a group, see pairwise debts,
and receive a minimal set of suggested transactions to settle all debts.

## Design

### How it works

**Member views group balances.** A member requests balances for a group.
The system calculates each member's net balance: total paid minus fair
share of all expenses. A positive balance means the member is owed money;
negative means they owe. The response lists every member with their net
balance.

**Member views pairwise debts.** A member requests the debt breakdown
for a group. The system returns a list of directed debts: who owes whom
and how much. Self-debts (payer owes themselves) are excluded. Only
non-zero debts appear.

**Member views settlement suggestions.** A member requests settlement
suggestions for a group. The system computes the minimum number of
transactions needed to settle all debts. Each suggestion names a payer,
a payee, and an amount. The algorithm minimizes transaction count, not
individual transaction size.

**Member records a settlement.** A member records a payment made to
settle a debt. The settlement has a payer, payee, and amount. Recording
a settlement affects subsequent balance calculations — it reduces the
payer's debt to the payee. Settlements are stored permanently, not as
modifications to expenses.

### Key decisions

**Equal split only.** All expenses are split equally among all group
members. Percentage-based and exact-amount splits add complexity without
core value for v1. This can be extended later without schema changes to
the balance system.

**Balances computed on read, not stored.** Balances are derived from
expenses and settlements at query time. No materialized balance table.
This avoids consistency bugs at the cost of computation, which is
negligible for expected group sizes (under 50 members, under 1000
expenses).

**Settlement is a first-class record, not an expense.** Settlements are
not modeled as expenses with special flags. They have different
semantics: an expense is split among the group, a settlement is between
two specific members. Separate storage prevents confusion.

**Debt simplification algorithm for settlement suggestions.** Use a
greedy algorithm: compute net balances, then repeatedly match the
largest creditor with the largest debtor. This minimizes transaction
count and is simple to implement. Optimal (graph-theoretic) solutions
are not worth the complexity for this use case.

### Constraints

- Single currency. All amounts are in the same unit. No conversion.
- No partial settlements in suggestions. Each suggestion settles the
  full amount between two members. Members can record partial settlements
  manually.
- Integer cents. All amounts stored and computed as integer cents to
  avoid floating-point errors.
- Rounding: when an expense does not divide evenly, the remainder cents
  are distributed one per member starting from the first member
  (alphabetical by name). This is deterministic and auditable.

### Edge cases

- **Empty group (no expenses):** All balances are zero. No debts. No
  settlement suggestions. Return empty arrays.
- **Single member:** Balance is zero (they paid themselves). No debts.
- **Zero-amount expense:** Allowed but contributes nothing to balances.
- **Member added after expenses exist:** New member is included in
  future expense splits only. Past expenses are not recalculated. This
  is consistent with "split among all current members at time of
  expense" — but note: the current schema does not record which members
  existed when an expense was created. For v1, all current members share
  all expenses. This is a known simplification.
- **All debts already settled:** Settlement suggestions return empty.
- **Negative expense amount:** Not allowed. Validation rejects amounts
  less than or equal to zero.

## Behavior Inventory

1. Retrieve per-member net balances for a group
2. Retrieve pairwise directed debts for a group
3. Retrieve settlement suggestions (minimal transactions) for a group
4. Record a settlement between two members in a group
5. Settlement affects subsequent balance and debt calculations
6. Reject expenses with non-positive amounts
7. Handle rounding for uneven splits deterministically

## Outstanding Questions

None.

---

## Context: Existing Codebase

Same as scenario-define.md. Express/Node/SQLite with groups, members,
and expenses CRUD. No balance logic, no settlements table.

## Grading Rubric

An evaluator agent scores each item as PASS / PARTIAL / FAIL.

### 1. Data Model

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Settlements table proposed | Table with payer_id, payee_id, group_id, amount, created_at (or equivalent) | Table proposed but missing key columns | No new table proposed, or settlements mixed into expenses |
| Column types specified | Integer cents for amount, foreign keys for member references, NOT NULL constraints | Types mentioned but incomplete | No types specified |
| Indexes identified | At least an index on group_id for the settlements table | Mentioned but not specific | No indexes discussed |
| Justifies table existence | Explains why settlements are a separate table, not a flag on expenses | Implied | No justification |
| No unnecessary tables | Does not propose a balances or debts table (since computed on read) | Proposes a table but notes it is optional | Proposes materialized balance tables contradicting the spec |

### 2. Capability Decomposition

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Capabilities are ordered by dependency | Balance calculation before settlement suggestions; settlement recording before "settlement affects balances" | Mostly ordered with 1 dependency inversion | No clear ordering or major inversions |
| Each capability delivers independent value | Each one answers "what can someone do now?" with a concrete action | Most do, but 1-2 are layer-only (e.g., "add settlements table" with no endpoint) | Multiple layer-only capabilities |
| Granularity is appropriate | 4-7 capabilities for this feature. Each is one user action or system behavior | Either too coarse (1-2 mega-capabilities) or too fine (10+ micro-steps) | Capabilities are architectural layers, not user-facing |
| Expense validation included | Rejecting non-positive amounts is a capability (or part of one) | Mentioned in notes but not a capability | Missing |
| Rounding logic addressed | Rounding behavior is part of the balance calculation capability or its own | Mentioned but not placed in a capability | Missing |

### 3. Behavior Traceability

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Every behavior mapped to a capability | All 7 behaviors from the inventory map to at least one capability | 5-6 mapped | Fewer than 5 mapped, or no mapping provided |
| No orphan capabilities | Every capability traces back to at least one behavior (or is explicitly identified as a technical necessity) | 1 orphan capability without justification | Multiple unexplained capabilities |

### 4. Technical Decisions

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Respects spec decisions | Does not override equal-split-only, compute-on-read, or separate settlements table | Overrides one decision with justification | Silently contradicts the spec |
| Algorithm approach specified | Names the greedy debt-simplification approach and describes it concretely | Mentions an algorithm but vaguely | No algorithm discussion |
| API shape proposed | Specific route names and response shapes for balance, debts, and settlement endpoints | Routes named but response shape unclear | No API design |

### 5. Implementation Notes

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Rounding rule specified | The deterministic rounding rule from the spec is restated with implementation guidance | Mentioned | Not addressed |
| Existing patterns referenced | Names specific files or patterns in the existing codebase to follow | References "existing patterns" generically | No mention of existing code |
| Testing strategy present | Identifies specific calculations or edge cases worth testing (rounding, zero members, settlement effect on balance) | Generic "test the endpoints" | No testing strategy |

### 6. Plan Completeness

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Handoff-ready for /execute | An agent reading only this plan could implement commit-by-commit without asking questions | 1-2 minor gaps | Major gaps requiring design decisions during implementation |
| Accounts for existing codebase | Builds on existing routes/patterns, does not propose rewriting existing CRUD | Mostly builds on existing code | Proposes rewriting existing functionality |
| Commit sequence is atomic | Each capability maps to one or more complete commits (not layer-by-layer) | Most commits are atomic | Commits are organized by layer (all migrations first, then all routes) |
