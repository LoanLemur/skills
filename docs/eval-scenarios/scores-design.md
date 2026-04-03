# Scorecard: /design — Balance Calculation Feature

## 1. Data Model — 4.5 / 5

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Settlements table proposed | PASS | Table fully specified with payer_id, payee_id, group_id, amount, created_at. |
| Column types specified | PARTIAL | Types present but amount is REAL rather than integer cents, which the spec requires. |
| Indexes identified | PASS | `CREATE INDEX idx_settlements_group_id ON settlements(group_id)` explicitly named. |
| Justifies table existence | PASS | Explains semantic difference from expenses directly. |
| No unnecessary tables | PASS | No balances or debts table; states balances computed on read. |

## 2. Capability Decomposition — 5.0 / 5

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Capabilities ordered by dependency | PASS | Validation → balances → debts → suggestions → recording → settlement effect; no inversions. |
| Each capability delivers independent value | PASS | Every capability is a concrete user action; none are layer-only. |
| Granularity appropriate | PASS | Six capabilities, all user-facing, within 4–7 range. |
| Expense validation included | PASS | Capability 1 is explicitly expense amount validation. |
| Rounding logic addressed | PASS | Capability 2 covers rounding with Math.floor and alphabetical remainder. |

## 3. Behavior Traceability — 2.0 / 2

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Every behavior mapped | PASS | All 7 behaviors mapped in traceability section. |
| No orphan capabilities | PASS | All capabilities trace to behaviors. |

## 4. Technical Decisions — 2.5 / 3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Respects spec decisions | PARTIAL | Uses REAL instead of integer cents, contradicting spec constraint (justified but still a deviation). |
| Algorithm approach specified | PASS | Greedy algorithm described concretely. |
| API shape proposed | PASS | All routes named with HTTP methods and response shapes. |

## 5. Implementation Notes — 3.0 / 3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Rounding rule specified | PASS | Exact pseudocode provided. |
| Existing patterns referenced | PASS | Specific files named throughout. |
| Testing strategy present | PASS | Ten specific test cases with expected values. |

## 6. Plan Completeness — 3.0 / 3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Handoff-ready for /execute | PASS | File names, method signatures, SQL schema, algorithm steps all present. |
| Accounts for existing codebase | PASS | Builds on existing patterns; REAL deviation framed as avoiding disruption. |
| Commit sequence atomic | PASS | Each capability is complete with route, model, and tests. |

## Summary

| Category | Score | Possible |
|----------|-------|----------|
| Data Model | 4.5 | 5 |
| Capability Decomposition | 5.0 | 5 |
| Behavior Traceability | 2.0 | 2 |
| Technical Decisions | 2.5 | 3 |
| Implementation Notes | 3.0 | 3 |
| Plan Completeness | 3.0 | 3 |
| **Total** | **20.0** | **21** |

**Overall: 95.2%**
