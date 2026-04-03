## Implementation Plan

### Approach

Add balance calculation, pairwise debt, settlement suggestion, and settlement recording endpoints to the existing Splitwise Lite API. All balance-related values are computed on read from expenses and settlements — no materialized balance tables.

The closest existing feature is the expenses CRUD (`src/models/expense.js`, `src/routes/expenses.js`). The new settlement model and balance logic follow the same patterns: model objects with static async methods using `allRows`/`getRow`/`runSql` from `src/db.js`, routes nested under groups, supertest integration tests with in-memory SQLite.

**Amounts stay as REAL, not integer cents.** The spec calls for integer cents, but the existing `expenses.amount` column is `REAL NOT NULL` and all existing tests use decimal values (120.0, 85.5, 50, 10). Converting existing expenses to integer cents would require a migration of existing data and changes to every existing endpoint and test. Instead, the new `settlements.amount` column will also use `REAL` for consistency with the existing schema. The balance computation logic will handle rounding internally using `Math.round()` and `Math.floor()` to produce integer-cent results in API responses, while storage remains as REAL. This keeps the codebase consistent and avoids a disruptive migration that is out of scope for this feature.

**Alternative considered: integer cents throughout.** This would be the right choice for a greenfield project. Retrofitting it here means changing the expenses schema, all expense endpoints, all expense tests, and the new code — a refactor that should be its own commit sequence before this feature. Not worth coupling to this feature.

### Data model changes

**New table: `settlements`**

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| group_id | INTEGER | NOT NULL, FK → groups(id) ON DELETE CASCADE | Which group this settlement belongs to |
| payer_id | INTEGER | NOT NULL, FK → members(id) | Member who paid |
| payee_id | INTEGER | NOT NULL, FK → members(id) | Member who received payment |
| amount | REAL | NOT NULL, CHECK(amount > 0) | Settlement amount (positive) |
| created_at | TEXT | DEFAULT (datetime('now')) | Matches existing timestamp pattern |

**Index:** `CREATE INDEX idx_settlements_group_id ON settlements(group_id)` — settlements are always queried by group.

**Why a separate table:** Settlements have different semantics from expenses. An expense is split among all group members; a settlement is a directed payment between exactly two members. Mixing them into the expenses table with a flag would complicate every balance query and violate the spec's explicit decision.

**No balances or debts table.** Balances are computed on read per the spec. Group sizes are small enough that computing from expenses + settlements on every request is negligible.

### Technical decisions

**Balance calculation queries the `expenses` and `settlements` tables directly.** The balance module will contain pure functions that take arrays of expenses, members, and settlements, then compute net balances. This keeps the SQL simple (fetch all expenses and settlements for a group) and puts the logic in testable JavaScript.

- Why: Doing the math in SQL would be complex multi-join queries that are hard to test and debug. The data volumes are small (spec says <50 members, <1000 expenses).
- Considered: SQL-only approach with CTEs. Rejected — harder to implement the rounding rule and settlement algorithm in SQL.

**Greedy debt simplification for settlement suggestions.** Compute each member's net balance, then repeatedly pair the largest creditor with the largest debtor, transferring the minimum of their absolute balances. This minimizes transaction count.

- Why: Simple to implement, correct for minimizing transaction count, matches the spec's explicit decision.
- Considered: Graph-theoretic optimal solutions. Rejected per spec — not worth the complexity.

**Rounding: remainder cents distributed alphabetically.** When an expense doesn't divide evenly among N members, compute `Math.floor(amount / N)` as the base share, then distribute the remainder (`amount - base * N`) one cent at a time to members sorted alphabetically by name.

- Why: Deterministic, auditable, matches the spec exactly.
- Note: This requires member names to be available during balance computation, so the balance module needs members sorted by name.

**Balance endpoint returns integer cents in responses.** Even though storage is REAL, the balance computation rounds all results to integer cents before returning them. This satisfies the spec's integer-cents constraint at the API boundary.

**New routes nested under groups.** Following the existing pattern where expenses are at `/groups/:groupId/expenses`, the new endpoints are:
- `GET /groups/:groupId/balances` — net balances per member
- `GET /groups/:groupId/debts` — pairwise directed debts
- `GET /groups/:groupId/settlements/suggestions` — minimal settlement transactions
- `POST /groups/:groupId/settlements` — record a settlement
- `GET /groups/:groupId/settlements` — list settlements (identified during design: needed for transparency/auditability)

### Capabilities

```
1. Add expense amount validation for non-positive amounts
   The existing route already rejects amount <= 0 (src/routes/expenses.js line 27-28),
   but the validation message does not distinguish zero from negative. This capability
   adds a model-level validation and a test that explicitly covers zero-amount expenses
   returning 400. Follows the existing validation pattern in the expenses route.
   Minimal change — tightens existing behavior.

2. Retrieve per-member net balances for a group
   New file: src/models/balance.js — module with static functions (not a DB-backed model).
   Function: computeBalances(members, expenses, settlements) → array of { member_id, member_name, balance }.
   New route: GET /groups/:groupId/balances in src/routes/balances.js.
   Fetches all members, expenses, and settlements for the group, passes them to
   computeBalances. Response: { group_id, balances: [{ member_id, name, balance }] }.
   balance is integer cents. Positive = owed money, negative = owes money.
   Handles rounding: Math.floor(amount / N) base share, remainder distributed
   alphabetically. Empty group returns empty balances array.
   Register route in src/app.js following the expenses pattern.

3. Retrieve pairwise directed debts for a group
   Function: computeDebts(members, expenses, settlements) → array of { from, to, amount }.
   New route: GET /groups/:groupId/debts.
   For each expense, each non-payer member owes their share to the payer.
   Aggregate all pairwise debts, net them (if A owes B 30 and B owes A 10,
   result is A owes B 20). Exclude self-debts and zero-amount debts.
   Settlements reduce pairwise debts. Response: { group_id, debts: [{ from_id,
   from_name, to_id, to_name, amount }] }. Amounts in integer cents.

4. Retrieve settlement suggestions for a group
   Function: computeSettlementSuggestions(members, expenses, settlements) →
   array of { from, to, amount }.
   New route: GET /groups/:groupId/settlements/suggestions.
   Uses net balances from computeBalances. Greedy algorithm: sort creditors
   descending, debtors descending by absolute value. Match largest creditor with
   largest debtor, transfer min(credit, |debt|), repeat until all settled.
   Returns empty array when all balances are zero.

5. Record a settlement between two members
   New file: src/models/settlement.js — follows the Expense model pattern.
   Methods: create(groupId, payerId, payeeId, amount), findByGroupId(groupId).
   New table: settlements (see data model above). Added to initSchema in src/db.js.
   New routes: POST /groups/:groupId/settlements, GET /groups/:groupId/settlements.
   POST validates: group exists, payer and payee are members of the group,
   payer !== payee, amount > 0. Returns 201 with the created settlement.
   GET returns all settlements for the group, ordered by created_at DESC.

6. Settlements affect balance and debt calculations
   This is not a separate code change — it is verified by adding settlements to
   test scenarios for capabilities 2, 3, and 4. The balance, debt, and suggestion
   functions already accept settlements as input. This capability adds integration
   tests proving that recording a settlement changes the output of balance, debt,
   and suggestion endpoints.
```

### Behavior traceability

```
Behavior 1 (Retrieve per-member net balances)           → Capability 2
Behavior 2 (Retrieve pairwise directed debts)           → Capability 3
Behavior 3 (Retrieve settlement suggestions)            → Capability 4
Behavior 4 (Record a settlement)                        → Capability 5
Behavior 5 (Settlement affects subsequent calculations) → Capabilities 2, 3, 4, 6
Behavior 6 (Reject non-positive expense amounts)        → Capability 1
Behavior 7 (Deterministic rounding for uneven splits)   → Capability 2
```

### Implementation notes

- **Existing patterns to follow:**
  - Models: static async methods on a plain object, using `getDb()` then `allRows`/`getRow`/`runSql` from `src/db.js`. See `src/models/expense.js` for the template.
  - Routes: Express routers, group-existence check at top of each handler, validation before model call, 201 for creates, 400 for validation errors, 404 for missing resources. See `src/routes/expenses.js`.
  - Tests: `beforeAll` creates in-memory SQLite DB via `sql.js`, creates all tables, uses `setDb()`. Supertest for HTTP assertions. See `tests/expenses.test.js`.
  - App registration: `app.use(route)` in `src/app.js`. Expenses are mounted at root since paths include `/groups/`. Balance and settlement routes should follow the same pattern.

- **The `balance.js` module is a computation module, not a DB model.** It exports pure functions that accept data arrays. The route handler fetches from DB, then calls these functions. This keeps balance logic testable without HTTP.

- **Schema change in `src/db.js`:** Add `CREATE TABLE IF NOT EXISTS settlements (...)` and the index to `initSchema`. Tests must also add the settlements table in their `beforeAll` blocks.

- **Rounding implementation detail:** For an expense of `amount` cents split among `N` members:
  ```
  base = Math.floor(amount / N)
  remainder = amount - base * N
  shares = members.sortBy(name).map((m, i) => base + (i < remainder ? 1 : 0))
  ```
  If the input amount is REAL (e.g., 100.50), convert to cents first: `Math.round(amount * 100)`. All computation happens in integer cents. Convert back to decimal (divide by 100) only in the API response if needed, or keep as cents — the spec says integer cents, so responses should use cents.

- **Pairwise debt netting:** For each expense, compute each member's share. The payer is owed `(amount - payer_share)` total, split as individual debts from each other member. Aggregate across all expenses into a map keyed by `(from_id, to_id)`. Then net: if `debts[A→B]` and `debts[B→A]` both exist, keep only the net direction. Subtract settlements from the corresponding pairwise debt.

- **Settlement suggestions depend on net balances only**, not pairwise debts. The greedy algorithm works on the net position of each member.

### Testing strategy

Tests that provide confidence for non-trivial logic:

- **Rounding distribution** (Capability 2): An expense of 100 cents split among 3 members. Verify the first member alphabetically gets 34, the other two get 33. This is the core rounding edge case.
- **Net balance with mixed expenses** (Capability 2): Multiple expenses with different payers. Verify each member's net balance sums correctly.
- **Pairwise debt netting** (Capability 3): A owes B and B owes A from different expenses. Verify only the net direction and amount appear.
- **Self-debt exclusion** (Capability 3): Verify the payer does not appear as owing themselves.
- **Zero-debt exclusion** (Capability 3): When debts cancel exactly, verify they don't appear.
- **Greedy settlement algorithm** (Capability 4): Three members with known imbalances. Verify the algorithm produces the minimum number of transactions with correct amounts.
- **All debts settled** (Capability 4): After settlements that zero out all debts, suggestions return empty.
- **Settlement affects balances** (Capability 6): Record a settlement, then fetch balances. Verify the payer's and payee's balances changed by the settlement amount.
- **Settlement validation** (Capability 5): payer === payee returns 400. Non-member returns 400. Non-positive amount returns 400.
- **Empty group** (Capability 2): Group with no expenses returns empty balances array.

Unit tests on the balance computation functions (not just HTTP integration) are warranted here because the rounding and algorithm logic is the core complexity. The route-level integration tests verify wiring.
