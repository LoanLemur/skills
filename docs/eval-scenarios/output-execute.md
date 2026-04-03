# /execute Output: Balance Calculation Feature

## Commit Log

```
$ git log main..HEAD --oneline
352ee21 feat: Settlements affect balance calculations
0b75b85 feat: Add settlement recording endpoint
73ef56a feat: Add settlement suggestion endpoint
70fd756 feat: Add pairwise debt calculation endpoint
1b7acbb feat: Add per-member balance calculation
```

## Test Status

All tests pass: 34 tests across 4 suites (balances, settlements, expenses, groups).

```
Test Suites: 4 passed, 4 total
Tests:       34 passed, 34 total
```

## Commit-to-Capability Mapping

| Commit | Capability |
|--------|-----------|
| 1b7acbb feat: Add per-member balance calculation | Cap 2: Per-member balance + rounding. Also covers Cap 1 implicitly (validation already existed). |
| 70fd756 feat: Add pairwise debt calculation endpoint | Cap 3: Pairwise debts via greedy algorithm |
| 73ef56a feat: Add settlement suggestion endpoint | Cap 4: Minimal settlement suggestions |
| 0b75b85 feat: Add settlement recording endpoint | Cap 5: POST settlements with validation + settlements table |
| 352ee21 feat: Settlements affect balance calculations | Cap 6: All three read endpoints incorporate settlements |

## Deviations from Plan

1. **Capability 1 (expense amount validation) was not a separate commit.**
   The existing codebase already validated `amount > 0` at the
   application level in `src/routes/expenses.js`. Adding a CHECK
   constraint to the existing `expenses` table is not straightforward
   in SQLite (requires table recreation) and provides no additional
   value since the app validation already rejects non-positive amounts.
   Skipped as already implemented.

2. **The plan references better-sqlite3 but the codebase uses sql.js.**
   All database access uses the async `getDb()` pattern with sql.js
   in-memory databases. Tests create their own in-memory databases via
   `initSqlJs()` and inject them via `setDb()`.

3. **Existing expense amounts are stored as REAL (float), not INTEGER.**
   The balance computation module uses `Math.round()` on expense
   amounts to convert to integer cents before splitting, maintaining
   integer arithmetic for all balance calculations. New settlement
   amounts are stored as INTEGER in the settlements table.

4. **The balance module is pure functions (no DB access).**
   As specified in the plan, `computeBalances`, `computeDebts`, and
   `computeSuggestions` are pure functions. Controllers query the
   database and pass data to these functions.

## Architecture

- `src/balances.js` — Pure computation module (computeBalances, computeDebts, computeSuggestions)
- `src/routes/balances.js` — GET /groups/:groupId/balances, /debts, /settlements/suggestions
- `src/routes/settlements.js` — POST /groups/:groupId/settlements
- `src/models/settlement.js` — Settlement model (findByGroupId, create)
- `src/db.js` — Updated schema with settlements table + index
- `tests/balances.test.js` — 13 tests covering balances, debts, suggestions, rounding, edge cases
- `tests/settlements.test.js` — 9 tests covering settlement CRUD + settlement effect on calculations
