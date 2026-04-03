# Scenario: /review — Balance Calculation Feature

## Input: Branch with Intentional Problems

The branch under review implements the balance calculation feature from
the spec in scenario-design.md. It has 5 commits, described below. The
evaluator should create this branch in the test repo before running
/review.

### Branch Setup Instructions

Create a branch `feature/balance-calculation` off main with 5 commits.
Each commit is described with its content and intentional problems.

---

### Commit 1: "refactor: Extract expense validation and add balance module"

**Content:**
- Adds input validation to the existing expense creation route
  (rejects amount <= 0)
- Also creates `src/balances.js` with the `computeBalances` function
- Adds a test for expense validation

**Intentional problem: Mixed commit.** This commit is a refactor
(adding validation to existing code) combined with a feature (creating
the balance module). These are two different categories per CLAUDE.md
commit conventions — refactors and features must never share a commit.

---

### Commit 2: "feat: Add balance and debt endpoints"

**Content:**
- Adds `GET /groups/:groupId/balances` endpoint
- Adds `GET /groups/:groupId/debts` endpoint
- Adds `computeDebts` to `src/balances.js`
- Implements the rounding rule for uneven splits
- Adds tests for both endpoints including a rounding test

**Intentional problem: None.** This commit is clean. It delivers two
related capabilities (balance viewing and debt viewing) which is
acceptable since debts are derived from balances and share the same
computation path.

---

### Commit 3: "fix stuff"

**Content:**
- Adds `GET /groups/:groupId/settlements/suggestions` endpoint
- Adds `computeSuggestions` (greedy algorithm) to `src/balances.js`
- Adds a test verifying suggestion count but not correctness

**Intentional problem: Bad commit message.** "fix stuff" is vague, not
imperative in a meaningful way, uses wrong prefix (fix: implies a bug
fix, this is a feature), and has no descriptive subject. Should be
something like `feat: Add settlement suggestion endpoint`.

---

### Commit 4: "feat: Add settlement recording"

**Content:**
- Creates the settlements table (migration or schema change)
- Adds `POST /groups/:groupId/settlements` endpoint
- Validates payer_id != payee_id and amount > 0
- Does NOT validate that payer and payee are members of the group
- Adds test for creating a settlement

**Intentional problem: Missing validation.** The endpoint does not
check that payer_id and payee_id are actual members of the specified
group. The spec requires this. A user could record a settlement between
members of different groups.

---

### Commit 5: "feat: Include settlements in balance calculations"

**Content:**
- Modifies `computeBalances` and `computeDebts` to factor in settlements
- Modifies `computeSuggestions` to use settlement-adjusted debts
- The `computeBalances` function does NOT handle the case where
  settlement amounts are zero (technically the CHECK constraint prevents
  this, but the function uses `amount / memberCount` without guarding
  against zero members)
- Does NOT add a test for the edge case: "what happens when all debts
  are already settled?" (spec edge case: should return empty suggestions)
- The spec says suggestions should return empty when all settled; the
  implementation returns `[{from: X, to: X, amount: 0}]` instead of
  filtering zeros

**Intentional problems:**
1. **Missing edge case test.** The spec explicitly lists "all debts
   already settled" as an edge case. No test covers it.
2. **Spec divergence.** The implementation returns zero-amount
   suggestions instead of an empty array when all debts are settled.
   The spec says: "Returns empty array when all settled."
3. **Zero-member division.** `computeBalances` divides by member count
   without guarding against zero members. The spec lists "empty group"
   as an edge case that should return an empty array, but the code
   would throw a division-by-zero error.

---

## Summary of Planted Problems

| # | Problem | Commit | Type |
|---|---------|--------|------|
| 1 | Refactor mixed with feature | 1 | Atomicity |
| 2 | Commit message "fix stuff" | 3 | Message quality |
| 3 | Missing group membership validation for settlements | 4 | Correctness |
| 4 | Missing test for "all settled" edge case | 5 | Test coverage |
| 5 | Zero-amount suggestions returned instead of empty array | 5 | Spec divergence |
| 6 | Division by zero on empty group | 5 | Correctness |

## Grading Rubric

An evaluator agent scores each item as PASS / PARTIAL / FAIL.

### 1. Catches Mixed Commit (Problem 1)

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Identifies the atomicity violation | Flags that commit 1 mixes a refactor (expense validation) with a feature (balance module) and recommends splitting | Notes the commit does multiple things but does not identify it as a category violation | Does not flag the mixed commit |
| Recommends a specific fix | Recommends splitting into two commits: (1) refactor adding validation, (2) feat adding balance module | Recommends splitting but vaguely | No fix recommended |

### 2. Catches Bad Commit Message (Problem 2)

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Flags "fix stuff" | Identifies the message as vague, wrong prefix, and non-descriptive | Flags it as vague but does not mention the wrong prefix | Does not flag the commit message |
| Suggests a replacement | Proposes a specific replacement like `feat: Add settlement suggestion endpoint` | Suggests improving but no specific replacement | No suggestion |

### 3. Catches Missing Validation (Problem 3)

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Identifies missing group membership check | Flags that settlement endpoint does not verify payer and payee belong to the group | Mentions validation could be improved but not specifically group membership | Does not identify the missing validation |
| Explains the consequence | States that members from different groups could create invalid settlements | Mentions a risk but vaguely | No consequence stated |

### 4. Catches Missing Edge Case Test (Problem 4)

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Identifies the missing "all settled" test | Flags that the spec's "all debts already settled" edge case has no test coverage | Notes test coverage could be better but not this specific case | Does not flag missing tests |

### 5. Catches Spec Divergence (Problem 5)

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Identifies zero-amount suggestions issue | Flags that the implementation returns zero-amount suggestions instead of empty array, and that this contradicts the spec | Notes the behavior but does not reference the spec | Does not identify the divergence |
| References the spec | Cites the spec's statement that suggestions should return empty when settled | Implies spec disagreement | No spec reference |

### 6. Catches Division-by-Zero Bug (Problem 6)

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Identifies the zero-member risk | Flags that `computeBalances` divides by member count without handling empty groups | Notes empty group handling could be improved | Does not identify the bug |
| References the spec | Notes the spec lists empty group as an edge case requiring empty array response | Mentions edge cases generally | No spec reference |

### 7. Quality of Review Process

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Suggests specific fixes | For each problem found, recommends a concrete action (split commit, reword message, add validation, add test, fix logic) | Some findings have fixes, others are observations | Findings are observations without recommended fixes |
| Verifies against spec | Explicitly cross-references the behavior inventory and edge cases from the spec | References the spec occasionally | Does not reference the spec |
| Does not manufacture false findings | Does not flag problems in the clean commit (2) that do not exist. Does not recommend unnecessary additions | 1-2 minor false findings | Multiple false findings or recommends significant unnecessary work |
| Reviews commit by commit | Walks through commits sequentially, evaluating each on its own terms | Reviews most commits individually | Reviews the branch as a blob without commit-level analysis |
