## Problem

Group members using Splitwise Lite can create expenses and record who paid, but there is no way to see the net financial position of each member within a group. Without balances, members cannot determine who owes whom or how much. Without settlement suggestions, groups must manually work out the minimum set of payments to square up — an error-prone process that grows combinatorially with group size.

The actors are group members who need to understand their financial standing and settle debts efficiently.

## Goal

After this ships, any group member can view per-member net balances (positive = owed money, negative = owes money), see a simplified list of who-owes-whom debts, and get a minimal set of suggested settlement transactions. Recording a settlement updates the balances accordingly.

## Design

### How it works

**Group member views balances**

A member requests the balance summary for their group. The system computes each member's net position by summing all expenses (each expense is split equally among all group members) and subtracting any settlements already recorded. The response is a mapping of member IDs to net amounts. Positive means the member is owed money; negative means they owe money. Zero means settled. If the group has no expenses, every member shows a zero balance.

**Group member views debts**

A member requests the simplified debt list. The system takes the net balances, separates debtors (negative balance) from creditors (positive balance), and pairs them using a greedy algorithm: match the largest debtor with the largest creditor, transfer the minimum of the two amounts, and repeat. This produces a minimal number of directed payments. The response is an array of `{from, to, amount}` objects. If all balances are zero, the array is empty.

**Group member requests settlement suggestions**

A member requests suggestions for how to settle up. The system runs the same greedy minimization algorithm as debts but filters out negligible amounts (below half a cent). The response is an array of `{from, to, amount}` objects representing the fewest transactions needed to zero out all balances.

**Group member records a settlement**

A member records an actual payment between two members. The system stores the settlement and it is factored into subsequent balance calculations. The payer and payee must be different members. The amount must be positive. After recording, balances reflect the settlement.

### Key decisions

**Equal splitting only — no custom splits, percentages, or shares**

Equal splitting covers the dominant use case (shared meals, trips, rent) and keeps the calculation straightforward. Custom splits (percentage-based, share-based, or itemized) add significant complexity to both the input validation and the balance algorithm. They should ship as a separate feature after equal splitting is proven.

Rejected: Percentage splits, share-based splits, itemized splits. These are valuable but each is a distinct feature with its own UI, validation, and edge cases.

**Greedy debt minimization rather than optimal (minimum-transaction) algorithm**

The greedy approach (sort debtors and creditors by amount, match largest pairs) produces near-optimal results and is simple to implement and reason about. The truly optimal minimum-transaction solution is NP-hard in the general case and requires subset-sum enumeration. For typical group sizes (2-20 members), greedy produces identical results to optimal in most cases.

Rejected: Optimal subset-sum algorithm. Unnecessary complexity for the group sizes this app targets.

**Settlements are append-only records, not edits to expenses**

Settlements are recorded as separate entities rather than modifying expense records. This preserves the full history of who paid for what and who settled with whom. A member can see both the original expenses and the settlements. Deleting or editing settlements is out of scope for this feature.

Rejected: Mutating expense records to reflect settlements. Destroys audit trail and makes it impossible to distinguish "Alice paid for dinner" from "Bob paid Alice back."

### Constraints

- **Currency**: Single currency only. All amounts are raw numbers with no currency identifier. Multi-currency support (conversion rates, display formatting) is out of scope.
- **Atomicity**: Balance computation is a read-only derivation — no stored balance state to corrupt. Settlement recording is a single insert. No partial-completion scenarios.
- **Scale**: The equal-split calculation iterates all expenses and settlements for a group on each request. This is adequate for groups with hundreds of expenses. For groups with thousands, a materialized balance approach would be needed — that is out of scope.
- **Rounding**: Amounts are rounded to two decimal places. When an expense doesn't split evenly (e.g., $100 among 3 people), rounding residuals should net to less than one cent across all members.
- **Expense splitting assumes all current group members share equally**. There is no concept of a member joining mid-trip and only sharing later expenses.

### Edge cases

**Group with no expenses**
Trigger: A member requests balances for a group that has members but no expenses.
User sees: All balances are zero. Debts array is empty. Suggestions array is empty.
System: Returns the zero-balance map and empty arrays.

**Group with one member**
Trigger: A single-member group has expenses (the member paid for everything and splits with themselves).
User sees: Balance is zero — the sole member owes nothing to themselves.
System: The share equals the full amount, so the net is zero. No debts or suggestions.

**Expense amount that doesn't divide evenly**
Trigger: $100 expense split among 3 members.
User sees: Balances that sum to within one cent of zero (rounding residual).
System: Each share is rounded to two decimal places. The sum of all balances may be off by up to one cent. This is acceptable.

**Self-settlement attempt**
Trigger: A member tries to record a settlement where payer and payee are the same person.
User sees: 400 error — payer and payee must be different.
System: Rejects the request before inserting.

**Settlement exceeding owed amount**
Trigger: Bob owes Alice $30 but records a $50 settlement.
User sees: The settlement is accepted. Alice's balance goes negative (she now owes Bob $20).
System: No cap is enforced. Overpayment is allowed — it simply flips the balance direction. This avoids complex "you can only settle up to what you owe" validation that would require computing balances inside the settlement write path.

**Member added after expenses exist**
Trigger: A new member joins a group that already has recorded expenses.
User sees: The new member has a zero share of past expenses. Only future expenses include them in the split.
System: Current implementation splits all expenses by the current member count, which means adding a member retroactively changes everyone's balance. This is a known limitation — expense splits should be snapshotted at creation time. This is deferred to a future feature.

**Deleted expense affecting balances**
Trigger: An expense is deleted after settlements have been recorded based on the original balances.
User sees: Balances update to reflect the deleted expense. Existing settlements remain, which may cause overpayment.
System: No warning is issued. Balances are recomputed from current data.

**Zero-amount settlement attempt**
Trigger: A member tries to record a settlement with amount zero or negative.
User sees: 400 error — amount must be positive.
System: The settlements table has a CHECK constraint enforcing amount > 0.

## Codebase Context

The app uses an Express REST API with an in-memory SQLite database (sql.js). Models follow an async static-method pattern (`Model.findById`, `Model.create`) wrapping raw SQL. Routes are nested REST style — expenses and settlements are scoped under `/groups/:groupId/`. The balance computation is a standalone module with pure functions that query expenses, settlements, and members, then derive balances in-memory. Tests use supertest with a fresh in-memory database per test suite. There is no authentication or authorization — all endpoints are open.

## Behavior Inventory

1. **Compute per-member balances**: GET request for a group returns a map of member ID to net balance (positive = owed, negative = owes), derived from all expenses split equally among members minus recorded settlements.
2. **Compute simplified debts**: GET request for a group returns an array of directed `{from, to, amount}` payments derived from net balances using greedy largest-first pairing.
3. **Compute settlement suggestions**: GET request for a group returns a minimal set of `{from, to, amount}` transactions to zero out all balances, filtering out sub-cent residuals.
4. **Record a settlement**: POST request records a payment from one member to another. Validates payer and payee are different, amount is positive, and both belong to the group. The settlement is factored into subsequent balance computations.
5. **List settlements for a group**: GET request returns all recorded settlements for a group, ordered by most recent first.
6. **Handle rounding residuals**: Balance computation rounds to two decimal places and ensures the sum of all member balances nets to less than one cent.
7. **Return empty/zero results for groups with no expenses**: All balance-related endpoints return zero balances and empty arrays when no expenses exist.

Deferred:
- Custom expense splits (percentage, shares, itemized) — separate feature
- Multi-currency support — separate feature
- Settlement editing/deletion — separate feature
- Snapshot expense splits at creation time (so adding members later doesn't retroactively change old splits) — separate feature
- Member-level expense participation (exclude members from specific expenses) — separate feature

## Outstanding Questions

1. Should adding a new member to a group retroactively change the equal split on existing expenses? The current implementation recomputes splits based on current membership, meaning adding a member changes everyone's historical balance. Snapshotting splits at expense creation time would be more correct but requires storing the participant list per expense.
2. Should there be a cap preventing settlements that exceed the amount owed? Currently overpayment is allowed and simply flips the balance. A cap would require computing balances in the settlement write path, adding complexity.
