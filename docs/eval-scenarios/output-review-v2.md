# Review: `feature/balance-calculation-review`

## Phase 1: Context

No issue or PR provided. Branch name contains no issue number. Reviewing
commits vs main with rigorous edge case audit.

```
git log main..HEAD --oneline
7de1387 feat: Include settlements in balance calculations
0309b55 feat: Add settlement recording
3ab7125 fix stuff
dc5a7d2 feat: Add balance and debt endpoints
ec8cabb refactor: Extract expense validation and add balance module
```

```
 src/app.js                |   5 ++-
 src/balances.js           |  72 +++++++++++++++++++++++++++++++
 src/db.js                 |  80 +++++------------------------------
 src/models/settlement.js  |  13 ++++++
 src/routes/balances.js    |  24 +++++++++++
 src/routes/expenses.js    |   6 ++-
 src/routes/settlements.js |  29 +++++++++++++
 tests/balances.test.js    |  71 +++++++++++++++++++++++++++++++
 tests/expenses.test.js    | 105 ++++++++++------------------------------------
 tests/groups.test.js      |  94 ++++++++++-------------------------------
 tests/settlements.test.js |  45 ++++++++++++++++++++
 11 files changed, 318 insertions(+), 226 deletions(-)
```

---

## Phase 2: Overview

### Branch narrative

This branch adds balance tracking and settlement recording to a
Splitwise-like expense-splitting app. The arc: extract a balance
computation module, expose balance/debt endpoints, add settlement
recording, then wire settlements into the balance calculations.

Reading the one-line log in sequence:

1. `refactor: Extract expense validation and add balance module` --
   "and" signals two things in one commit. Problematic.
2. `feat: Add balance and debt endpoints` -- clear.
3. `fix stuff` -- uninformative, wrong prefix, violates message
   conventions.
4. `feat: Add settlement recording` -- clear.
5. `feat: Include settlements in balance calculations` -- clear.

The subjects do not tell a clean story. Commit 3 ("fix stuff") is
a grab-bag that adds a new feature (`computeSuggestions` +
`/settlements/suggestions` endpoint) disguised as a fix.

### Atomicity scan

- **Commit 1** (`ec8cabb`) mixes a refactor (extract expense
  validation, rewrite tests) with a new module (`balances.js`). The
  balance module is not used by any route yet -- it is a layer-only
  addition. Two concerns in one commit.
- **Commit 2** (`dc5a7d2`) is well-scoped: adds routes, wires the
  module, includes tests.
- **Commit 3** (`3ab7125`) is labeled "fix stuff" but actually adds
  the `computeSuggestions` function and a new endpoint
  (`/settlements/suggestions`). This is a feature, not a fix. It also
  makes cosmetic changes to existing tests. Multiple concerns.
- **Commit 4** (`0309b55`) adds settlement recording (model, routes,
  tests) but also rewrites `src/db.js` formatting and rewrites
  `tests/groups.test.js` formatting. The db.js rewrite adds the
  settlements table to `initSchema` -- that belongs here -- but the
  whitespace-only reformatting of the entire file is a separate
  concern. Same with groups.test.js reformatting. Also removes
  assertions from balances.test.js and expenses.test.js (weakening
  existing tests) which is unrelated to settlement recording.
- **Commit 5** (`7de1387`) is well-scoped: wires settlements into
  balance calculations with a test.

### Ordering

Commit 3 adds `/settlements/suggestions` before settlements even
exist (commit 4). The suggestions endpoint should come after settlement
recording. Reordering: 1, 2, 4, 5, then suggestions as a final commit.

### Structural recommendations

1. Split commit 1: refactor (validation + test cleanup) separate from
   balance module introduction (which should merge into commit 2).
2. Commit 3 should be split: the suggestions feature is its own commit,
   and the test cosmetic changes should fold into wherever they belong
   (or be dropped).
3. The db.js reformatting in commit 4 should be a separate refactor
   commit or folded into commit 1's refactor.
4. Stop weakening test assertions across unrelated commits.

---

## Phase 3: Commit-by-commit Review

### Commit 1: `ec8cabb` -- refactor: Extract expense validation and add balance module

Delivers: description validation on expense creation, plus a new
`balances.js` module (unused until commit 2).

```diff
+async function computeBalances(groupId) {
+  const db = await getDb();
+  const members = allRows(db, 'SELECT id, name FROM members WHERE group_id = ?', [groupId]);
+  const expenses = allRows(db, 'SELECT * FROM expenses WHERE group_id = ?', [groupId]);
+
+  const balances = {};
+  members.forEach(m => { balances[m.id] = 0; });
+
+  for (const expense of expenses) {
+    const share = expense.amount / members.length;
```

**Edge case audit -- division by zero:** `members.length` is the
divisor. If a group has zero members, this divides by zero producing
`Infinity`. A group with expenses but no members is unlikely but not
impossible (members could be deleted). This should guard against empty
members or return early.

**Edge case audit -- empty collections:** If `members` is empty, the
`balances` object stays empty and the loop over expenses still runs,
producing `Infinity` values via the division. If `expenses` is empty,
the function returns all-zero balances, which is correct.

```diff
+    balances[expense.paid_by] += expense.amount - share;
+    members.forEach(m => {
+      if (m.id !== expense.paid_by) {
+        balances[m.id] -= share;
+      }
+    });
```

**Edge case audit -- conditional:** `if (m.id !== expense.paid_by)` --
the other branch (when they match) is handled by the line above which
adds `amount - share` to the payer. Correct.

**Edge case audit -- filter (implicit):** If `expense.paid_by` refers
to a member not in this group's member list (data integrity issue),
`balances[expense.paid_by]` would be `undefined + number = NaN`. The
foreign key constraint mitigates this in production, but deleted
members could trigger it.

```diff
+  if (typeof description !== 'string' || description.trim().length === 0) {
+    return res.status(400).json({ error: 'description must be a non-empty string' });
+  }
```

Good validation addition. Guards against `null`, `undefined`, numeric
descriptions, and whitespace-only strings.

**Atomicity:** The commit message says "Extract expense validation
**and** add balance module." Two things. The balance module is not
consumed by any route in this commit -- it is dead code at this point
in history. The validation extraction and test rewrite are a coherent
refactor; the balance module belongs in commit 2 where it is first
used.

**Test changes:** The test rewrite compresses formatting but also
removes several assertions: `res.body.description` check,
`res.body.amount` check, `res.body.paid_by_name` check. These
provided confidence that the response shape was correct. Removing them
weakens coverage without justification. The new test for
whitespace-only description is good -- it covers the new validation.

**Commit message:** Subject exceeds the spirit of "one thing" --
should be two commits. The `refactor:` prefix is inaccurate for the
balance module addition (that is new code, not restructuring).

**Findings:**
1. `computeBalances` should return early when `members` is empty to
   avoid division by zero.
2. Split this commit: refactor (validation + test cleanup) in one,
   balance module in another (or fold into commit 2).
3. Do not remove response-shape assertions from existing tests without
   reason.

---

### Commit 2: `dc5a7d2` -- feat: Add balance and debt endpoints

Delivers: `GET /groups/:groupId/balances` and
`GET /groups/:groupId/debts` endpoints with `computeDebts` algorithm.

```diff
+async function computeDebts(groupId) {
+  const balances = await computeBalances(groupId);
+  const debtors = [], creditors = [];
+  for (const [id, bal] of Object.entries(balances)) {
+    if (bal < 0) debtors.push({ id: Number(id), amount: -bal });
+    else if (bal > 0) creditors.push({ id: Number(id), amount: bal });
+  }
```

**Edge case audit -- filter:** When `bal === 0`, the member is neither
debtor nor creditor and is silently skipped. Correct -- zero-balance
members do not need to transact.

**Edge case audit -- empty collections:** If all balances are zero (no
expenses), both `debtors` and `creditors` are empty. The while loop
does not execute. Returns empty array. Correct.

```diff
+  while (i < debtors.length && j < creditors.length) {
+    const payment = Math.min(debtors[i].amount, creditors[j].amount);
+    const rounded = Math.round(payment * 100) / 100;
+    if (rounded > 0) debts.push({ from: debtors[i].id, to: creditors[j].id, amount: rounded });
+    debtors[i].amount -= payment;
+    creditors[j].amount -= payment;
+    if (debtors[i].amount < 0.005) i++;
+    if (creditors[j].amount < 0.005) j++;
+  }
```

The greedy debt simplification algorithm is a standard approach for
minimizing transactions. The 0.005 threshold avoids floating-point
dust preventing loop termination.

**Edge case audit -- both `if` statements advancing:** If
`debtors[i].amount === creditors[j].amount`, both `i` and `j`
advance. This is correct -- the debt is fully settled on both sides.

**Edge case audit -- `rounded` guard:** If `payment` is between 0 and
0.005 (exclusive), `rounded` is 0, and no debt is pushed. But
`debtors[i].amount -= payment` still reduces the amount, and the
threshold check advances the index. Correct behavior -- sub-cent dust
is absorbed.

```diff
+router.get('/groups/:groupId/balances', async (req, res) => {
+  const group = await Group.findById(req.params.groupId);
+  if (!group) return res.status(404).json({ error: 'group not found' });
+  res.json(await computeBalances(group.id));
+});
```

Clean. GET endpoints with no side effects, proper 404 handling.

**Responsibility:** The balance computation logic lives in
`src/balances.js`, not in the controller. The controller is skinny --
it validates the group exists and delegates. Good.

**DRY concern:** The group-existence check is duplicated in both route
handlers. Minor -- extracting middleware for two routes would be
premature.

**Test coverage:** Tests check specific balance values for a known
scenario (Alice pays 60, Bob pays 30, 3 members). The rounding test
verifies balances sum to near-zero. The 404 test covers the guard.
Tests are simpler than the code. Good.

**Findings:**
1. No defects found in this commit's code.
2. The commit is well-scoped -- one capability (balance/debt
   endpoints) fully delivered with tests.

---

### Commit 3: `3ab7125` -- fix stuff

Delivers: `computeSuggestions` function and
`GET /groups/:groupId/settlements/suggestions` endpoint.

```diff
+async function computeSuggestions(groupId) {
+  return computeDebts(groupId);
+}
```

`computeSuggestions` is a trivial wrapper around `computeDebts`. It
adds a layer of indirection with zero value. This is replaced in
commit 5 with a real implementation, so at this point in history it is
dead weight that will be thrown away two commits later.

```diff
+router.get('/groups/:groupId/settlements/suggestions', async (req, res) => {
+  const group = await Group.findById(req.params.groupId);
+  if (!group) return res.status(404).json({ error: 'group not found' });
+  res.json(await computeSuggestions(group.id));
+});
```

Duplicates the group-existence pattern. Fine mechanically.

**Edge case audit:** `computeSuggestions` delegates entirely to
`computeDebts` -- same edge cases apply (empty group, zero balances).
No new logic to audit.

```diff
+describe('Settlement suggestions', () => {
+  test('returns suggestions array', async () => {
+    const res = await request(app).get(`/groups/${groupId}/settlements/suggestions`);
+    expect(res.status).toBe(200);
+    expect(Array.isArray(res.body)).toBe(true);
+    expect(res.body.length).toBeGreaterThanOrEqual(0);
+  });
+});
```

**Test confidence:** `expect(res.body.length).toBeGreaterThanOrEqual(0)`
asserts nothing -- every array satisfies this. The test verifies only
that the endpoint returns 200 and an array. It does not verify the
content of any suggestion. Low confidence.

The commit also makes cosmetic changes to the rounding test (shorter
description string, inlined variable). These are unrelated to the
suggestions feature.

**Commit message:** "fix stuff" is the worst possible commit message.
Wrong prefix (`fix:` implies a bug fix), no description of what
changed, violates every message convention. The actual content is a
feature addition.

**Atomicity:** This should not exist as a separate commit. The
suggestions endpoint is a feature that should be introduced after
settlements exist (commit 4) and with a real implementation (commit 5).
Introducing a throwaway wrapper first creates a commit that will be
immediately rewritten.

**Findings:**
1. Rename or drop this commit entirely. The suggestions feature should
   be a single commit after settlement recording, with the real
   implementation from commit 5.
2. The test asserts nothing meaningful --
   `toBeGreaterThanOrEqual(0)` on an array length is vacuous.
3. "fix stuff" violates every commit message convention.

---

### Commit 4: `0309b55` -- feat: Add settlement recording

Delivers: Settlement model, `GET/POST /groups/:groupId/settlements`
endpoints, settlements table in schema.

```diff
+const Settlement = {
+  async findByGroupId(groupId) {
+    const db = await getDb();
+    return allRows(db, 'SELECT * FROM settlements WHERE group_id = ? ORDER BY created_at DESC', [groupId]);
+  },
+  async create(groupId, payerId, payeeId, amount) {
+    const db = await getDb();
+    const result = runSql(db, 'INSERT INTO settlements (group_id, payer_id, payee_id, amount) VALUES (?, ?, ?, ?)', [groupId, payerId, payeeId, amount]);
+    return getRow(db, 'SELECT * FROM settlements WHERE id = ?', [result.lastInsertRowid]);
+  },
+};
```

Clean model. Parameterized queries prevent SQL injection. The
`CHECK(amount > 0)` constraint in the schema provides a database-level
safety net matching the route validation.

**Edge case audit -- `findByGroupId` with no settlements:** Returns
empty array from `allRows`. Correct.

```diff
+router.post('/groups/:groupId/settlements', async (req, res) => {
+  const group = await Group.findById(req.params.groupId);
+  if (!group) return res.status(404).json({ error: 'group not found' });
+  const { payer_id, payee_id, amount } = req.body;
+  if (!payer_id || !payee_id || !amount) {
+    return res.status(400).json({ error: 'payer_id, payee_id, and amount are required' });
+  }
```

**Edge case audit -- conditional:** `!amount` is falsy for `0`, which
is correctly rejected (settlements must be positive). But `!payer_id`
is also falsy for `0` -- if member IDs could be 0 (SQLite
AUTOINCREMENT starts at 1, so unlikely but not guaranteed), this
would incorrectly reject valid input. `!payee_id` has the same issue.

```diff
+  if (payer_id === payee_id) {
+    return res.status(400).json({ error: 'payer and payee must be different' });
+  }
+  if (typeof amount !== 'number' || amount <= 0) {
+    return res.status(400).json({ error: 'amount must be a positive number' });
+  }
```

Good validations. But missing: **no check that payer_id and payee_id
are valid members of this group.** A client could post arbitrary
member IDs from another group. The foreign key constraint would catch
cross-group references only if the member ID does not exist at all --
it would not catch a valid member from a different group. The expense
route validates `member.group_id !== group.id`; settlements should do
the same.

**Responsibility:** Validation logic lives in the controller.
Consistent with the expense route's pattern in this codebase.

**Unrelated changes in this commit:**

The `src/db.js` diff is 80 lines of reformatting (collapsing
multiline SQL to single lines, removing comments, compressing helper
functions). This is a style refactor that has nothing to do with
settlement recording. It should be a separate refactor commit.

The `tests/groups.test.js` diff removes 50 lines of formatting and
several assertions (`res.body.name`, `res.body.id`,
`res.body.group_id`, `res.body.length`). These are not related to
settlements. Weakening existing tests in an unrelated commit is a red
flag.

The `tests/balances.test.js` diff removes the `from` and `to`
assertions from the debt test and removes `expect(res.status)` checks.
These assertions were verifying the debt algorithm's correctness --
removing them reduces test confidence. Unrelated to settlement
recording.

Similarly `tests/expenses.test.js` removes the `description` assertion
and compresses the delete test.

**Findings:**
1. **Missing validation:** Settlement creation does not verify that
   `payer_id` and `payee_id` belong to the specified group. Add
   member-existence checks matching the expense route pattern.
2. **Unrelated changes:** db.js reformatting, groups/balances/expenses
   test reformatting, and assertion removals should not be in this
   commit. Split them out or revert the unrelated hunks.
3. **Weakened tests:** Removing `from`/`to` assertions from the debt
   test, removing `description` assertion from the expense test, and
   removing `res.body.name` from the group test all reduce coverage
   for no stated reason.

---

### Commit 5: `7de1387` -- feat: Include settlements in balance calculations

Delivers: Settlements now affect balance calculations. Suggestions
endpoint returns remaining debts after accounting for settlements.

```diff
+  const settlements = allRows(db, 'SELECT * FROM settlements WHERE group_id = ?', [groupId]);
```

**Edge case audit -- empty collection:** If no settlements exist,
returns empty array, the loop does not execute, balances are
expense-only. Correct.

```diff
+  for (const settlement of settlements) {
+    balances[settlement.payer_id] = (balances[settlement.payer_id] || 0) + settlement.amount;
+    balances[settlement.payee_id] = (balances[settlement.payee_id] || 0) - settlement.amount;
+  }
```

The `|| 0` fallback handles the case where a settlement references a
member ID not in the `balances` object. This could happen if a member
was deleted after recording a settlement. Defensive coding, though it
introduces a key into `balances` that was not in the original members
list -- downstream consumers iterating over `balances` would see a
ghost member.

**Correctness:** Settlement semantics: payer gives money to payee.
From a balance perspective, the payer's debt decreases (balance goes
up) and the payee's credit decreases (balance goes down). The signs
are correct.

```diff
+async function computeSuggestions(groupId) {
+  const balances = await computeBalances(groupId);
+  const debtors = [], creditors = [];
+  for (const [id, bal] of Object.entries(balances)) {
+    if (bal < -0.005) debtors.push({ id: Number(id), amount: -bal });
+    else if (bal > 0.005) creditors.push({ id: Number(id), amount: bal });
+  }
```

This replaces the trivial wrapper from commit 3. The threshold changed
from `0` (in `computeDebts`) to `0.005` (here in `computeSuggestions`).
The 0.005 threshold filters out floating-point dust before entering
the algorithm -- good improvement.

**DRY concern:** `computeSuggestions` and `computeDebts` now contain
nearly identical greedy matching algorithms. The only differences:
(1) suggestions uses 0.005 threshold for filtering, debts uses 0;
(2) output field names differ (`suggestions` vs `debts` array name).
This is real duplication. `computeDebts` should either call
`computeSuggestions` or both should delegate to a shared function.
Alternatively, `computeDebts` may not even need to exist separately
since suggestions supersede it once settlements exist.

**Edge case audit -- division:** No new divisions introduced.

**Edge case audit -- empty after filter:** If all balances are within
the 0.005 threshold (everyone nearly settled), both arrays are empty,
while loop does not execute, returns empty suggestions. Correct.

**Edge case audit -- `amount` being 0 after rounding:** If
`Math.min(debtors[i].amount, creditors[j].amount)` is between 0 and
0.005, `Math.round(... * 100) / 100` produces 0. A suggestion with
`amount: 0` would be pushed. Unlike `computeDebts` which has
`if (rounded > 0)`, `computeSuggestions` does not guard against this.
This is a bug -- zero-amount suggestions can appear in the output.

```diff
+  test('balances reflect settlements', async () => {
+    const g = await request(app).post('/groups').send({ name: 'Settle Test' });
+    const gid = g.body.id;
+    const a = await request(app).post(`/groups/${gid}/members`).send({ name: 'A' });
+    const b = await request(app).post(`/groups/${gid}/members`).send({ name: 'B' });
+    await request(app).post(`/groups/${gid}/expenses`).send({ description: 'Food', amount: 100, paid_by: a.body.id });
+    await request(app).post(`/groups/${gid}/settlements`).send({ payer_id: b.body.id, payee_id: a.body.id, amount: 30 });
+    const res = await request(app).get(`/groups/${gid}/balances`);
+    expect(res.body[a.body.id]).toBe(20);
+    expect(res.body[b.body.id]).toBe(-20);
+  });
```

Good test. Verifies the math: A pays 100 for 2 people (each owes 50),
so A's balance is +50. B's balance is -50. After B settles 30 to A:
A = 50 - 30 = 20, B = -50 + 30 = -20. Assertions match.

**Findings:**
1. **Bug:** `computeSuggestions` can produce zero-amount suggestions.
   Add the `if (amount > 0)` guard that `computeDebts` already has.
2. **DRY:** `computeDebts` and `computeSuggestions` are near-identical.
   Extract the greedy matching into a shared function, or have
   `computeDebts` delegate to the shared logic.

---

## Phase 4: Wrap-up

### Final narrative check

```
ec8cabb refactor: Extract expense validation and add balance module
dc5a7d2 feat: Add balance and debt endpoints
3ab7125 fix stuff
0309b55 feat: Add settlement recording
7de1387 feat: Include settlements in balance calculations
```

The narrative is broken by commit 3 ("fix stuff") which is a mislabeled
feature addition. The ordering places suggestions before settlements
exist. The first commit conflates refactoring with new module creation.

**Recommended commit sequence after surgery:**

1. `refactor: Add description validation to expense creation`
2. `feat: Add balance and debt endpoints`
3. `feat: Add settlement recording`
4. `feat: Include settlements in balance and suggestion calculations`

### Cross-commit consistency

- **Test assertion style drift:** Early commits have explicit
  assertions (`expect(res.body.description).toBe(...)`) but later
  commits progressively remove them. By the end of the branch, several
  test files have weaker assertions than they started with. This is a
  regression in test confidence across the branch.
- **Validation pattern inconsistency:** The expense route validates
  that `paid_by` is a member of the group. The settlement route does
  not validate that `payer_id` and `payee_id` are members of the
  group. Same pattern, inconsistent application.
- **db.js formatting:** The wholesale reformatting of db.js in commit 4
  is unrelated to settlements and makes the diff harder to review.
  This should be a separate refactor commit (or not done at all).
- **Duplication:** `computeDebts` and `computeSuggestions` contain the
  same greedy algorithm. This duplication was introduced across commits
  2 and 5 and should be consolidated.

### Summary of findings

| # | Severity | Commit | Finding |
|---|----------|--------|---------|
| 1 | Bug | ec8cabb, dc5a7d2 | `computeBalances` divides by `members.length` without guarding against zero members |
| 2 | Bug | 7de1387 | `computeSuggestions` can emit zero-amount suggestions (missing `> 0` guard present in `computeDebts`) |
| 3 | Missing validation | 0309b55 | Settlement creation does not verify payer/payee are members of the group |
| 4 | Atomicity | ec8cabb | Mixes refactor (validation) with new module (balances.js) |
| 5 | Atomicity | 3ab7125 | Mislabeled feature ("fix stuff"), adds suggestions endpoint as throwaway code |
| 6 | Atomicity | 0309b55 | Bundles unrelated db.js and test reformatting with settlement feature |
| 7 | DRY | dc5a7d2 + 7de1387 | `computeDebts` and `computeSuggestions` duplicate the greedy matching algorithm |
| 8 | Test quality | 0309b55 | Removes meaningful assertions from existing tests (debt from/to, expense description, group name) without justification |
| 9 | Commit message | 3ab7125 | "fix stuff" violates all message conventions |
| 10 | Ordering | 3ab7125 | Suggestions endpoint introduced before settlements exist |

### Test and lint status

All 26 tests pass. No linter configured in the project.
