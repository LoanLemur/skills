# Scenario: /execute — Balance Calculation Feature

## Input: Completed /design Plan

The following is a realistic implementation plan (what /design should
produce). Provide this as the GitHub issue body — appended after the
spec from scenario-design.md — when testing /execute.

---

## Implementation Plan

### Approach

Compute balances on read from expenses and settlements. Add a
settlements table for recording payments between members. Add three
read endpoints (balances, debts, suggestions) and one write endpoint
(record settlement). Follow existing Express route and controller
patterns.

The existing codebase uses express.Router per resource, SQLite via
better-sqlite3, and a simple controller-per-file pattern. New routes
follow the same structure.

### Data model changes

**settlements** table:

| Column | Type | Constraints |
|--------|------|-------------|
| id | INTEGER | PRIMARY KEY AUTOINCREMENT |
| group_id | INTEGER | NOT NULL, REFERENCES groups(id) |
| payer_id | INTEGER | NOT NULL, REFERENCES members(id) |
| payee_id | INTEGER | NOT NULL, REFERENCES members(id) |
| amount | INTEGER | NOT NULL, CHECK(amount > 0) |
| created_at | TEXT | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

- `amount` is in integer cents.
- Index on `group_id`.
- `payer_id` != `payee_id` enforced at application level.

**expenses** table modification:

- Add CHECK(amount > 0) constraint. Existing data assumed valid.

### Technical decisions

**Express route structure.** Balance, debt, and suggestion endpoints
are nested under groups: `GET /groups/:groupId/balances`,
`GET /groups/:groupId/debts`, `GET /groups/:groupId/settlements/suggestions`.
Settlements CRUD: `POST /groups/:groupId/settlements`. This follows the
existing nesting pattern (`/groups/:groupId/members`,
`/groups/:groupId/expenses`).

**Balance calculation in a shared module.** The balance computation
(sum paid minus fair share) is used by three endpoints. Extract to a
`balances.js` module that exports pure functions: `computeBalances`,
`computeDebts`, `computeSuggestions`. Controllers call these functions
with data from the database. This keeps controllers thin and logic
testable.

**Greedy debt simplification.** Compute net balances, sort creditors
descending, debtors ascending. Match largest creditor with largest
debtor, transfer the minimum of the two amounts, repeat until settled.

**Rounding rule.** When splitting amount X among N members: each share
is `Math.floor(X / N)`. Remainder `X % N` cents are distributed one
each to the first R members sorted alphabetically by name. Implement in
the balance computation module.

### Capabilities

```
1. Expense amount validation
   Add CHECK constraint and application-level validation rejecting
   expenses with amount <= 0. Returns 400 with error message.
   Follows existing error handling pattern in expense routes.

2. Per-member balance calculation endpoint
   GET /groups/:groupId/balances returns each member's net balance
   (total paid minus fair share of all expenses). Uses the rounding
   rule for uneven splits. Response: array of {member_id, name, balance}
   where balance is in integer cents (positive = owed, negative = owes).
   Empty group returns empty array.

3. Pairwise debt calculation endpoint
   GET /groups/:groupId/debts returns directed debts between members.
   Response: array of {from_id, from_name, to_id, to_name, amount}.
   Excludes self-debts and zero amounts. Empty group returns empty array.

4. Settlement suggestion endpoint
   GET /groups/:groupId/settlements/suggestions returns minimal
   transactions to settle all debts. Uses greedy algorithm on net
   balances. Response: array of {from_id, from_name, to_id, to_name,
   amount}. Returns empty array when all settled.

5. Record a settlement
   POST /groups/:groupId/settlements with {payer_id, payee_id, amount}.
   Creates a settlement record. Validates: payer and payee are different
   members of the group, amount > 0. Returns 201 with the settlement.

6. Settlements affect balance calculations
   Modify balance computation to subtract settlements from pairwise
   debts. A settlement from A to B reduces A's debt to B. This affects
   all three read endpoints. No new routes — modifies the balance
   module.
```

### Behavior traceability

```
Behavior 1 (per-member balances)     → Capability 2
Behavior 2 (pairwise debts)          → Capability 3
Behavior 3 (settlement suggestions)  → Capability 4
Behavior 4 (record settlement)       → Capability 5
Behavior 5 (settlement affects calc) → Capability 6
Behavior 6 (reject non-positive)     → Capability 1
Behavior 7 (rounding)                → Capability 2
```

### Implementation notes

- Follow the existing pattern: each route file exports a router,
  mounted in the main app file.
- Error responses: `{ error: "message" }` with appropriate HTTP status.
  Match existing patterns in expense and member routes.
- All amounts in integer cents. No floating point anywhere.
- The balance module should be pure functions (no database access).
  Controllers query the database and pass data to balance functions.

### Testing strategy

Tests worth writing:

- **Rounding edge case**: $10.00 (1000 cents) split among 3 members.
  Verify two members get 333 and one gets 334, allocated
  deterministically.
- **Settlement effect**: After recording a settlement, verify balances
  and debts reflect it correctly.
- **Greedy algorithm correctness**: Group with 3+ members and multiple
  expenses. Verify suggestions actually settle all debts (net balances
  go to zero if all suggestions are applied).
- **Zero-amount / empty group**: Verify empty arrays returned, not
  errors.

---

## Context: Test Repo

The test repo is a working Express/Node/SQLite app at
`splitwise-lite/`. It has:

- `src/app.js` — Express app setup, mounts routers
- `src/db.js` — SQLite connection via better-sqlite3
- `src/routes/groups.js` — Group CRUD
- `src/routes/members.js` — Member CRUD (nested under groups)
- `src/routes/expenses.js` — Expense CRUD (nested under groups)
- `src/middleware/` — Error handling
- `tests/` — Existing tests using Jest and supertest
- `package.json` — Dependencies include express, better-sqlite3, jest,
  supertest

## Grading Rubric

An evaluator agent scores each item as PASS / PARTIAL / FAIL. The
evaluator should run `git log main..HEAD --oneline` and inspect each
commit.

### 1. Commit Correspondence

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Each planned capability has a commit | All 6 capabilities are represented in the commit history (may be split further, which is fine) | 4-5 capabilities covered | 3 or fewer, or capabilities are missing without explanation |
| No unplanned work | Every commit traces to a planned capability or is a justified refactor split. Deviations are documented as issue comments | 1 minor unplanned commit | Significant unplanned work with no documentation |

### 2. Commit Atomicity

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Each commit is one logical change | Migration + route + controller + tests for one capability in one commit | Most commits are atomic, 1 mixes concerns | Multiple commits mix refactors with features or split atomic units across commits |
| No layer-only commits | No commit that is just "add migration" or "add tests" without the corresponding code | 1 layer-only commit | Multiple layer-only commits |
| Complete commits | Each commit includes everything it needs — no later commit fixes an oversight from an earlier one (fixup commits are acceptable if squashed) | 1 minor incompleteness | Commits that depend on later commits to work |

### 3. Commit Messages

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Imperative present tense | All messages use imperative ("Add balance endpoint" not "Added" or "Adding") | Most do | Multiple non-imperative messages |
| Conventional commit prefix | All use feat:, fix:, refactor:, test:, chore: appropriately | Most do | Missing or wrong prefixes |
| Subject under 50 chars | All subjects fit in 50 characters including prefix | Most do | Multiple long subjects |
| Subject says what, body says why | Subjects describe the capability delivered. Bodies (if present) explain motivation, not implementation | Subjects are clear, bodies absent (acceptable for obvious changes) | Subjects are vague ("fix stuff", "update code") |

### 4. Test Quality

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Rounding test exists | A test verifies the deterministic rounding behavior (e.g., 1000 cents / 3 members) | Test exists but does not verify the specific rounding allocation | No rounding test |
| Settlement effect test exists | A test verifies that recording a settlement changes subsequent balance/debt calculations | Test exists but is superficial | No test for settlement effect |
| Algorithm test exists | A test verifies the greedy settlement suggestions produce correct results (applying all suggestions settles all debts) | Test exists but only checks count, not correctness | No algorithm test |
| Edge case tests exist | Tests for empty group and/or zero-amount scenarios | 1 edge case tested | No edge case tests |
| Tests are simpler than code | Tests are straightforward: set up data, call endpoint, assert result. No complex test infrastructure | Most tests are simple | Tests are harder to understand than the code they test |

### 5. Implementation Correctness

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Balances are computed correctly | Endpoint returns correct net balances for a group with multiple expenses and members | Mostly correct but rounding is wrong | Incorrect balances |
| Debts are computed correctly | Pairwise debts are correct, self-debts excluded, zero amounts excluded | Mostly correct | Incorrect or includes self-debts |
| Suggestions minimize transactions | Greedy algorithm produces correct minimal settlement set | Suggestions settle debts but are not minimal | Algorithm is broken |
| Settlements are recorded | POST endpoint creates a settlement with proper validation (different members, positive amount, members in group) | Creates settlement but validation is incomplete | Endpoint does not work |
| Settlements affect calculations | After recording a settlement, all three read endpoints reflect it | Affects balances but not debts or suggestions | Settlements are not factored into calculations |
| Integer cents throughout | No floating-point math anywhere in balance/debt/settlement calculations | Floating point used but results are rounded to integers | Floating-point errors possible in output |

### 6. Follows Existing Patterns

| Criterion | PASS | PARTIAL | FAIL |
|-----------|------|---------|------|
| Route structure matches | New routes follow the same Express router pattern as existing group/member/expense routes | Mostly matches | Different pattern (e.g., all routes in one file) |
| Error handling matches | Error responses use the same format and status codes as existing endpoints | Format matches but status codes differ | Custom error format |
| Tests follow existing style | New tests use the same Jest/supertest patterns as existing tests | Mostly match | Different test framework or pattern |
