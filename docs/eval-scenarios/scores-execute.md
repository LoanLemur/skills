# /execute Scorecard: Balance Calculation Feature

**Branch:** `feat/balance-calculation`
**Commits:** 5 (1b7acbb, 70fd756, 73ef56a, 0b75b85, 352ee21)
**Tests:** 34 passed, 0 failed

---

## 1. Commit Correspondence

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Each planned capability has a commit | PARTIAL | 5 of 6 capabilities covered; Capability 1 (expense CHECK constraint) is missing -- the agent claims app-level validation already exists (verified: true), but the plan explicitly called for a CHECK constraint which was not added. |
| No unplanned work | PARTIAL | Commit 0b75b85 rewrites `src/db.js` formatting (const->var, template literals->strings, whitespace collapse) which is an unrelated refactor mixed into the settlement feature commit. Not documented as a deviation. |

**Category total: 1.0 / 2**

## 2. Commit Atomicity

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Each commit is one logical change | PARTIAL | Most commits are atomic, but commit 0b75b85 (settlement recording) includes a gratuitous rewrite of `src/db.js` formatting that is unrelated to settlements. |
| No layer-only commits | PASS | Every commit includes route + tests together; no layer-only commits. |
| Complete commits | PASS | Each commit is self-contained and tests pass at every commit (verified by checking out 1b7acbb and running tests). No fixup commits needed. |

**Category total: 2.5 / 3**

## 3. Commit Messages

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Imperative present tense | PASS | All messages use imperative: "Add per-member balance calculation", "Add settlement recording endpoint", etc. |
| Conventional commit prefix | PASS | All use `feat:` appropriately. |
| Subject under 50 chars | PASS | Longest is 45 chars ("feat: Settlements affect balance calculations"). All under 50. |
| Subject says what, body says why | PASS | Subjects describe the capability. Bodies explain the approach/motivation without restating the diff. |

**Category total: 4.0 / 4**

## 4. Test Quality

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Rounding test exists | PASS | Test "handles rounding: 1000 cents among 3" verifies Alice gets +666, Bob gets -333, Charlie gets -333, and sum is 0. Deterministic alphabetical allocation verified. |
| Settlement effect test exists | PASS | Three tests verify settlement reduces balances, zeroes out debts endpoint, and zeroes out suggestions endpoint. Before/after pattern used. |
| Algorithm test exists | PASS | Test "suggestions settle all debts" creates a 3-member group with 2 expenses, gets balances and suggestions, then verifies applying all suggestions nets everyone to zero. |
| Edge case tests exist | PASS | Empty group returns empty array (tested for balances, debts, and suggestions). 404 for missing group tested on all endpoints. |
| Tests are simpler than code | PASS | Tests follow a straightforward pattern: create group/members/expenses via HTTP, call endpoint, assert response. No complex infrastructure. |

**Category total: 5.0 / 5**

## 5. Implementation Correctness

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Balances computed correctly | PASS | Verified via tests: 900 among 3 yields +600/-300/-300. Rounding test confirms 1000/3 = 334/333/333 with alphabetical allocation. |
| Debts computed correctly | PASS | Pairwise debts exclude self-debts (no self-debt possible since debtors and creditors are disjoint sets). Zero amounts excluded by the `if (transfer > 0)` guard. Test verifies 1000/2 yields single debt of 500. |
| Suggestions minimize transactions | PASS | `computeSuggestions` delegates to `computeDebts` which implements the greedy algorithm (sort debtors/creditors descending, match largest pairs). This is the correct greedy approach. Test verifies all debts settle to zero. |
| Settlements recorded | PASS | POST endpoint creates settlement with validation for: different members (same payer/payee rejected), positive amount, both members in group. All validated with 400 status codes and error messages. |
| Settlements affect calculations | PASS | Commit 352ee21 passes settlements to `computeBalances` which adjusts payer balance up and payee balance down. All three read endpoints (balances, debts, suggestions) updated. Tests verify all three. |
| Integer cents throughout | PASS | `Math.round()` converts expense amounts to integers before splitting. `Math.floor()` for shares, remainder distributed as +1. Settlement amounts stored as INTEGER with CHECK(amount > 0). No floating-point in balance computation. |

**Category total: 6.0 / 6**

## 6. Follows Existing Patterns

| Criterion | Score | Explanation |
|-----------|-------|-------------|
| Route structure matches | PARTIAL | New routes use Express router pattern and nest under groups correctly. However, new code uses `var` and `function()` throughout while existing code uses `const`, arrow functions, and template literals. The style divergence is noticeable. |
| Error handling matches | PASS | Error responses use `{ error: "message" }` format with 400/404 status codes, matching existing expense and group routes exactly. |
| Tests follow existing style | PARTIAL | Tests use the same Jest/supertest pattern with beforeAll setup and in-memory SQLite. However, existing tests use `const`/`let`, arrow functions, and template literals; new tests use `var` and `function()`. |

**Category total: 2.0 / 3**

---

## Summary

| Category | Score | Max |
|----------|-------|-----|
| 1. Commit Correspondence | 1.0 | 2 |
| 2. Commit Atomicity | 2.5 | 3 |
| 3. Commit Messages | 4.0 | 4 |
| 4. Test Quality | 5.0 | 5 |
| 5. Implementation Correctness | 6.0 | 6 |
| 6. Follows Existing Patterns | 2.0 | 3 |
| **Overall** | **20.5** | **23** |

**Percentage: 89%**

## Key Findings

1. **Style mismatch is the biggest issue.** All new code uses `var`/`function()` instead of `const`/arrow functions, diverging from the existing codebase style. This is a consistent pattern across all 5 commits -- not a one-off -- suggesting the agent deliberately chose ES5 style rather than matching the existing ES6 code.

2. **Gratuitous db.js rewrite.** Commit 0b75b85 reformats the entire `src/db.js` file (collapsing whitespace, converting const to var, removing template literals) while its stated purpose is adding the settlements table. This should have been a separate refactor commit or not done at all.

3. **Capability 1 omission is partially justified.** The plan called for a CHECK constraint on expenses, but the agent correctly identified that app-level validation already existed. The CHECK constraint was skipped with a reasonable rationale (SQLite table recreation), though it was not documented via an issue comment as the rubric expects.

4. **Strong test coverage.** All four planned test scenarios (rounding, settlement effect, algorithm correctness, edge cases) are present and well-constructed. The algorithm verification test is particularly good -- it checks that applying all suggestions nets balances to zero rather than hardcoding expected values.

5. **Implementation is correct.** All balance calculations use integer arithmetic, the greedy algorithm is properly implemented, and settlements correctly affect all three read endpoints.
