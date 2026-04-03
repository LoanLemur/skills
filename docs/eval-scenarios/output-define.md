## Problem

Splitwise Lite lets users create groups, add members, and record expenses with a designated payer. But there is no way to see the net effect of those expenses: who owes whom, and by how much. Users must mentally reconstruct balances by scanning the expense list. There is also no guidance on how to settle up efficiently — in a group of N people, naive pairwise settlement can require up to N*(N-1)/2 transactions when far fewer are needed.

## Goal

After this ships, a user can request the current balances for any group and receive a per-member net balance (positive = owed money, negative = owes money). They can also request a settlement plan that minimizes the number of transactions needed to zero out all balances.

## Design

### How it works

**Group member viewing balances**

A user sends `GET /groups/:groupId/balances`. The system retrieves all expenses for the group, computes each member's net balance (amount paid minus equal share of each expense they participated in), and returns an array of member objects with their net balance. A positive balance means the group owes that member; a negative balance means that member owes the group.

All expenses are split equally among all current group members. The response includes each member's id, name, and balance rounded to two decimal places.

If the group has no expenses, all balances are zero. If the group has no members, the endpoint returns an empty array.

**Group member requesting settlement suggestions**

A user sends `GET /groups/:groupId/settlements`. The system computes net balances (same as above), then calculates the minimum set of transactions to settle all debts. Each settlement entry specifies a `from` member (debtor), a `to` member (creditor), and an `amount`.

The algorithm greedily matches the largest debtor with the largest creditor, reducing both balances, and repeats until all balances are zero (within a floating-point tolerance). This produces at most N-1 transactions for N members with non-zero balances.

The response is an array of settlement objects. If all balances are already zero, the array is empty.

### Key decisions

**Equal splits only — no custom split ratios or per-expense participant selection.** Equal splitting among all group members is the right starting point because the existing expense model has no concept of "participants" on an expense — every expense belongs to the group and has a single payer. Supporting custom splits would require a new expense_participants join table and changes to expense creation, which is a separate feature. This keeps the scope minimal while delivering the core value.

**Balances computed on read, not stored.** Balances are derived from the expense table at query time rather than maintained as a running total. This avoids dual-write consistency issues (balance getting out of sync with expenses when expenses are created or deleted). The expense table is small per-group, so query-time aggregation is fast enough. If performance becomes an issue, materialized balances can be added later without changing the API contract.

**Settlement uses a greedy algorithm, not optimal minimum transactions.** The greedy approach (match largest debtor to largest creditor) produces at most N-1 transactions and is simple to implement and reason about. Finding the true minimum number of transactions is NP-hard in the general case. The greedy result is good enough for typical group sizes (3-10 people) and often produces the optimal result anyway.

**Two separate endpoints rather than one combined endpoint.** Balances and settlements serve different purposes — a user may want to see "where do I stand" without being told how to settle. Keeping them separate follows the existing REST pattern in the codebase and avoids forcing clients to parse a larger response when they only need one piece.

### Constraints

- The existing expense model splits every expense across all group members equally. There is no per-expense participant list. Balance calculation must assume equal splits among all members.
- Balances must account for expenses that have been deleted (they simply won't appear in the query). There is no soft-delete; `DELETE /expenses/:id` removes the row.
- Floating-point arithmetic on currency amounts can produce rounding artifacts. Balances and settlement amounts should be rounded to two decimal places in the response.
- The sql.js in-memory database means all data is ephemeral. This doesn't affect the feature but is a constraint for testing — tests must set up their own data.

### Edge cases

**Group with no expenses.** `GET /groups/:groupId/balances` returns all members with a balance of 0. `GET /groups/:groupId/settlements` returns an empty array.

**Group with no members.** Both endpoints return empty arrays. No error — this is a valid state.

**Group not found.** Both endpoints return 404 with `{ error: 'group not found' }`, consistent with existing routes.

**Single member in group.** That member's balance is always 0 (they pay and owe the full amount). Settlements array is empty.

**Expense amount not evenly divisible by member count.** Per-member share is calculated as `amount / memberCount` and the resulting balances are rounded to two decimal places at response time. Small rounding remainders (fractions of a cent) are acceptable and will appear in the settlement suggestions.

**All expenses paid by one person.** That person has a large positive balance; everyone else has equal negative balances. Settlement suggestions direct all payments to that person.

**Rounding produces non-zero dust after settlement.** The greedy algorithm should treat balances within 0.01 of zero as settled. This prevents generating settlement transactions for sub-cent amounts.

## Codebase Context

The API follows a consistent pattern: models in `src/models/` expose async CRUD methods using the `db.js` helpers (`allRows`, `getRow`, `runSql`). Routes in `src/routes/` validate inputs, call model methods, and return JSON responses. Tests use supertest with an in-memory sql.js database initialized in `beforeAll`.

Key patterns to follow:
- **Route nesting:** Group sub-resources use `/groups/:groupId/...` (see `src/routes/expenses.js`).
- **Error handling:** 404 for missing groups, 400 for validation errors, consistent `{ error: '...' }` shape.
- **Model layer:** Models are plain objects with async methods, not classes. They call `getDb()` internally.
- **No middleware or auth.** All routes are public.

The expense model already joins with members to include `paid_by_name`. Balance calculation will query the same tables (expenses and members) but aggregate rather than list.

There are no existing balance or settlement concepts anywhere in the codebase. This is entirely new functionality.

## Behavior Inventory

1. `GET /groups/:groupId/balances` returns an array of `{ member_id, member_name, balance }` objects representing each member's net balance, computed from all group expenses split equally among all group members.
2. `GET /groups/:groupId/balances` returns 404 with `{ error: 'group not found' }` when the group does not exist.
3. `GET /groups/:groupId/balances` returns an empty array when the group has no members.
4. `GET /groups/:groupId/balances` returns all balances as 0 when the group has no expenses.
5. `GET /groups/:groupId/settlements` returns an array of `{ from, to, amount }` objects representing the minimum set of transactions to settle all debts, where `from` and `to` include member id and name.
6. `GET /groups/:groupId/settlements` returns 404 with `{ error: 'group not found' }` when the group does not exist.
7. `GET /groups/:groupId/settlements` returns an empty array when all balances are zero or the group has no members/expenses.
8. Settlement amounts and balances are rounded to two decimal places in all responses.
9. The settlement algorithm treats balances within 0.01 of zero as fully settled, producing no transaction for sub-cent remainders.

## Outstanding Questions

None. The scope is constrained to equal splits with read-only derived data, which avoids the design questions that would arise from custom splits or recorded settlements.
