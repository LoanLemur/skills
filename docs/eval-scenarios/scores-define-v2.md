# Scorecard: /define v2 — Balance Calculation Feature

## 1. Problem Statement — 3/3 (unchanged)
All PASS.

## 2. Behavior Inventory — 6/6 (up from 5/6)
- Pairwise debts: PASS (was PARTIAL — now a distinct behavior)
- Settlement recording: PASS (was PARTIAL — now explicitly covered)

## 3. Edge Cases — 4.5/6 (up from 3/6)
- Empty group: PASS
- Zero-amount expense: FAIL (still missing)
- Rounding errors: PASS
- Single-member group: PASS
- Member added after expenses: PASS (was FAIL)
- Self-payment in pairwise view: PARTIAL (was FAIL — self-settlement blocked but pairwise exclusion not explicit)

## 4. Constraints — 3/3 (up from 1/3)
- Currency scope: PASS (was FAIL)
- Split type scope: PASS
- Settlement scope: PASS (was FAIL)

## 5. Key Decisions — 3/3 (unchanged)
All PASS.

## 6. Spec Completeness — 2/3 (unchanged)
- Handoff-ready: PASS
- Free of implementation details: FAIL (codebase context still leaks sql.js, supertest, module patterns)
- Rejected alternatives: PASS

## Summary

| Category | v1 | v2 | Possible |
|----------|-----|-----|----------|
| Problem Statement | 3 | 3 | 3 |
| Behavior Inventory | 5 | 6 | 6 |
| Edge Cases | 3 | 4.5 | 6 |
| Constraints | 1 | 3 | 3 |
| Key Decisions | 3 | 3 | 3 |
| Spec Completeness | 2 | 2 | 3 |
| **Total** | **17** | **21.5** | **24** |

**v1: 70.8% → v2: 89.6% (+18.8pp)**

### Remaining gaps
- Zero-amount expense edge case still missing
- Implementation details still leak into Codebase Context
