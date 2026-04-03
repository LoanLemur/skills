# Review Branch Setup Complete

Branch `feature/balance-calculation-review` created at `/tmp/skill-test-repo/` with 5 commits:

```
7de1387 feat: Include settlements in balance calculations
0309b55 feat: Add settlement recording
3ab7125 fix stuff
dc5a7d2 feat: Add balance and debt endpoints
ec8cabb refactor: Extract expense validation and add balance module
```

All 26 tests pass (4 suites: groups, expenses, balances, settlements).

## Planted Problems

| # | Problem | Commit | Verified |
|---|---------|--------|----------|
| 1 | Refactor (expense validation) mixed with feature (balance module) in one commit | ec8cabb | Yes - src/balances.js created + src/routes/expenses.js validation added in same commit |
| 2 | Commit message "fix stuff" - wrong prefix, vague, non-descriptive | 3ab7125 | Yes |
| 3 | Missing group membership validation for settlements | 0309b55 | Yes - no Member import or lookup in src/routes/settlements.js |
| 4 | Missing test for "all debts settled" edge case | 7de1387 | Yes - no such test exists |
| 5 | Zero-amount suggestions returned instead of empty array | 7de1387 | Yes - computeSuggestions does not filter zero amounts |
| 6 | Division by zero on empty groups (members.length) | 7de1387 | Yes - `expense.amount / members.length` with no guard |
