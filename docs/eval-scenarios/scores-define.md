# Scorecard: /define — Balance Calculation Feature

## 1. Problem Statement — 3/3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| States what is broken or missing today | PASS | Names the gap: expenses exist but no way to see net balances or settle debts. |
| Names the actors | PASS | Names "group members" as primary actors. |
| States the goal | PASS | Single unambiguous goal sentence. |

## 2. Behavior Inventory — 5/6

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Numbered list of discrete behaviors | PASS | Nine behaviors, each with trigger, state, and outcome. |
| Covers balance viewing | PASS | Behavior 1 covers per-member net balances. |
| Covers pairwise debts | PARTIAL | Settlements show from/to pairs but no distinct "who owes whom" pairwise view. |
| Covers settlement suggestion | PASS | Behavior 5 covers minimal transactions. |
| Covers settlement recording | PARTIAL | Read-only scope implies deferral but never explicitly names or defers recording. |
| Behaviors are decomposed | PASS | All nine use specific verbs; none are vague. |

## 3. Edge Cases — 3/6

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Empty group | PASS | Both sub-cases addressed. |
| Zero-amount expense | FAIL | Not mentioned. |
| Rounding errors | PASS | Addressed with deterministic rule. |
| Single-member group | PASS | Explicitly addressed. |
| Member added after expenses exist | FAIL | Not mentioned. |
| Self-payment in pairwise view | FAIL | Not mentioned. |

## 4. Constraints — 1/3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Currency scope | FAIL | Single-currency assumption never stated. |
| Split type scope | PASS | Equal-split only stated with rationale. |
| Settlement scope | FAIL | Partial vs exact settlements unaddressed. |

## 5. Key Decisions — 3/3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Split type decision | PASS | Equal-split only with rationale. |
| Settlement algorithm | PASS | Greedy matching with rationale. |
| Balance calculation timing | PASS | Computed on read with rationale. |

## 6. Spec Completeness — 2/3

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Handoff-ready for /design | PASS | Enough detail for implementation plan without questions. |
| Free of implementation details | FAIL | Codebase Context names file paths, DB helpers, internal patterns — HOW not WHAT. |
| Rejected alternatives captured | PASS | Four rejected alternatives documented with rationale. |

## Summary

| Category | Score | Possible |
|----------|-------|----------|
| Problem Statement | 3 | 3 |
| Behavior Inventory | 5 | 6 |
| Edge Cases | 3 | 6 |
| Constraints | 1 | 3 |
| Key Decisions | 3 | 3 |
| Spec Completeness | 2 | 3 |
| **Total** | **17** | **24** |

**Overall: 70.8%**

### Gaps to address in skill
- 3 edge cases missing (zero-amount, member added after expenses, self-payment)
- Currency and settlement scope constraints missing
- Codebase Context leaks implementation details into a WHAT spec
