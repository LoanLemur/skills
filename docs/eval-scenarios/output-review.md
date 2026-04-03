# Review: `feature/balance-calculation-review`

## Phase 1: Context

```
7de1387 feat: Include settlements in balance calculations
0309b55 feat: Add settlement recording
3ab7125 fix stuff
dc5a7d2 feat: Add balance and debt endpoints
ec8cabb refactor: Extract expense validation and add balance module
```

11 files changed, +318 / -226. No issue or PR provided; behavior coverage
cannot be checked. The code speaks for itself.

---

## Phase 2: Overview

### Branch narrative

This branch adds balance calculation to a Splitwise Lite API. The arc:
extract a balance computation module, expose balance/debt endpoints,
add settlement recording, then wire settlements into the balance math.

### Commit narrative evaluation

Reading the one-line log in order:

1. `refactor: Extract expense validation and add balance module`
2. `feat: Add balance and debt endpoints`
3. `fix stuff`
4. `feat: Add settlement recording`
5. `feat: Include settlements in balance calculations`

**Problems:**

- **Commit 1 is not a refactor.** The subject says "refactor" but it
  introduces a brand-new `balances.js` module (new behavior) AND adds
  expense validation logic AND rewrites the test suite. A refactor must
  have zero behavior change with existing tests passing unmodified. This
  commit adds validation, creates new code, and rewrites tests -- it is
  at least two commits: a refactor of tests, and a feature adding
  validation + the balance module.
- **Commit 3 ("fix stuff") is an unacceptable commit message.** It gives
  zero information. Reading the diff, it adds a `computeSuggestions`
  function, a new `/settlements/suggestions` endpoint, and a test. This
  is a feature, not a fix. The subject should be something like
  `feat: Add settlement suggestion endpoint`.
- **Commit 1 mixes concerns.** Expense validation (new behavior),
  balance module creation (new code for later use), and test rewriting
  (chore) are three separate logical changes.
- **The balance module in commit 1 is dead code.** Nothing calls
  `computeBalances` until commit 2. The module, its queries, and its
  logic cannot be verified or used after commit 1. This violates
  atomicity: deliver the module with its first consumer.

### Atomicity scan

| Commit | Atomic? | Issue |
|--------|---------|-------|
| ec8cabb | No | Mixes refactor (test rewrite), feature (validation), and creates dead code (balance module with no consumer) |
| dc5a7d2 | Yes | Delivers balance + debt endpoints with tests. Should also include the balance module from commit 1. |
| 3ab7125 | Yes (content) | Delivers suggestions endpoint + test. But message is wrong and it also tweaks unrelated test assertions from commit 2. |
| 0309b55 | Borderline | Delivers settlements CRUD. But also rewrites `db.js` formatting (chore) and rewrites `groups.test.js` (unrelated). |
| 7de1387 | Yes | Wires settlements into balance calculations with a covering test. |

### Ordering

The ordering is logical in concept (compute -> expose -> settle ->
integrate). But the dead-code problem in commit 1 means the balance
module should be folded into commit 2.

### Structural recommendations

1. Squash the balance module from commit 1 into commit 2.
2. Split commit 1's remaining changes: test reformatting as a chore
   commit, expense validation as a feat commit.
3. Reword commit 3 to describe what it actually does.
4. The `db.js` reformatting and `groups.test.js` rewrite in commit 4
   should be a separate chore commit or folded into the test-reformat
   commit.

---

## Phase 3: Commit-by-Commit Review

### Commit 1: ec8cabb `refactor: Extract expense validation and add balance module`

This commit delivers: expense description validation in the expenses
route, a new `balances.js` module, and a rewrite of the expense test
suite.

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
+    balances[expense.paid_by] += expense.amount - share;
+    members.forEach(m => {
+      if (m.id !== expense.paid_by) {
+        balances[m.id] -= share;
+      }
+    });
+  }
```

The balance computation assumes equal splits across all group members.
This is reasonable for a Splitwise Lite, but worth noting: `SELECT *
FROM members` fetches all members, and the split denominator is
`members.length`. If a member was added after an expense was created,
they retroactively share that expense. This may be intentional (the
current group state defines splits) but is a domain assumption worth
flagging.

The `SELECT *` on expenses fetches all columns when only `amount` and
`paid_by` are needed. Minor, but `SELECT amount, paid_by` would be
more precise.

```diff
+  if (typeof description !== 'string' || description.trim().length === 0) {
+    return res.status(400).json({ error: 'description must be a non-empty string' });
+  }
```

Good validation. Trimming on save (`description.trim()`) is also
correct.

```diff
-  test('POST /groups/:groupId/expenses creates an expense', async () => {
+  test('creates an expense', async () => {
```

The test rewrite removes the route prefix from test names (good -- less
noise) but also removes assertions. The old test asserted
`res.body.description` and `res.body.paid_by_name`; the new test only
checks status 201. That weakens test confidence: the test now proves
the endpoint returns 201 but not that it actually created the right
data. The `description` assertion at minimum should remain.

The old `test('POST /groups/:groupId/expenses validates positive amount')`
tested `amount: -10`. The new test `'rejects zero amount'` tests
`amount: 0`. Both are valid edge cases, but the rename implies the test
only covers zero, not negative. The validation (`amount <= 0`) covers
both, so this is fine functionally, but the test name is narrower than
the behavior it tests.

**Atomicity violation (primary finding):** This commit has `refactor:`
prefix but introduces new behavior (description validation, new
balance module). The balance module is dead code at this point -- nothing
imports it. This should be split: (1) test reformatting as a chore, (2)
expense validation as a feat, (3) balance module should move to commit 2.

**Commit message:** Wrong prefix (`refactor:` for a commit containing
new features). Subject contains "and" which usually signals a
multi-concern commit. Over 50 characters.

`[reviewed] ec8cabb | [pending] dc5a7d2, 3ab7125, 0309b55, 7de1387`

---

### Commit 2: dc5a7d2 `feat: Add balance and debt endpoints`

This commit delivers: GET `/groups/:groupId/balances` and GET
`/groups/:groupId/debts` endpoints, the `computeDebts` function, route
registration in `app.js`, and tests.

```diff
+async function computeDebts(groupId) {
+  const balances = await computeBalances(groupId);
+  const debtors = [], creditors = [];
+  for (const [id, bal] of Object.entries(balances)) {
+    if (bal < 0) debtors.push({ id: Number(id), amount: -bal });
+    else if (bal > 0) creditors.push({ id: Number(id), amount: bal });
+  }
+  debtors.sort((a, b) => b.amount - a.amount);
+  creditors.sort((a, b) => b.amount - a.amount);
+  const debts = [];
+  let i = 0, j = 0;
+  while (i < debtors.length && j < creditors.length) {
+    const payment = Math.min(debtors[i].amount, creditors[j].amount);
+    const rounded = Math.round(payment * 100) / 100;
+    if (rounded > 0) debts.push({ from: debtors[i].id, to: creditors[j].id, amount: rounded });
+    debtors[i].amount -= payment;
+    creditors[j].amount -= payment;
+    if (debtors[i].amount < 0.005) i++;
+    if (creditors[j].amount < 0.005) j++;
+  }
+  return debts;
+}
```

This is a greedy debt simplification algorithm. It pairs the largest
debtor with the largest creditor, which minimizes the number of
transactions. The epsilon threshold of 0.005 for "close enough to zero"
is reasonable for currency rounding.

**Correctness concern:** `computeDebts` uses `bal < 0` as the threshold
for debtors, but the loop exit uses `< 0.005`. This means a balance of
exactly 0 is excluded (correct), but a balance of -0.001 would be
included as a debtor, then the loop would immediately skip it because
`0.001 < 0.005`. This is harmless but inconsistent -- the entry
threshold and exit threshold should match. `computeSuggestions` in the
final commit fixes this by using `-0.005` as the entry threshold.

```diff
+router.get('/groups/:groupId/balances', async (req, res) => {
+  const group = await Group.findById(req.params.groupId);
+  if (!group) return res.status(404).json({ error: 'group not found' });
+  res.json(await computeBalances(group.id));
+});
```

Clean. The group-existence check pattern is consistent with the expenses
route.

```diff
+  test('returns per-member balances', async () => {
+    const res = await request(app).get(`/groups/${groupId}/balances`);
+    expect(res.status).toBe(200);
+    expect(res.body[aliceId]).toBe(30);
+    expect(res.body[bobId]).toBe(0);
+    expect(res.body[carolId]).toBe(-30);
+  });
```

Good test. Setup creates a 3-member group with known expenses (Alice
pays 60, Bob pays 30), and the assertions verify exact balances. The
math checks out: Alice paid 60, her share is 30 (90/3), so she's owed
30. Bob paid 30, his share is 30, so he's at 0. Carol paid nothing,
owes 30.

The rounding test is valuable -- it verifies that balances sum to
approximately zero even with uneven splits.

Also removes some blank lines from `balances.js` (whitespace-only
changes mixed with feature work -- minor but technically a chore mixed
in).

**Commit message:** Good. Accurate, imperative, under 50 chars, correct
prefix.

`[reviewed] ec8cabb, dc5a7d2 | [pending] 3ab7125, 0309b55, 7de1387`

---

### Commit 3: 3ab7125 `fix stuff`

This commit delivers: a `computeSuggestions` function (alias for
`computeDebts`), a GET `/groups/:groupId/settlements/suggestions`
endpoint, and a test. It also tweaks the rounding test and an existing
debt test.

```diff
+async function computeSuggestions(groupId) {
+  return computeDebts(groupId);
+}
```

This is a function that wraps another function with zero added behavior.
At this point it is pure indirection. The comment in commit 5 reveals
the intent: `computeSuggestions` will later diverge from `computeDebts`
to account for settlements. But at this commit, it is dead weight. If
the intent is to create a separate function for future divergence, that
divergence should arrive in the same commit as the function.

```diff
+router.get('/groups/:groupId/settlements/suggestions', async (req, res) => {
+  const group = await Group.findById(req.params.groupId);
+  if (!group) return res.status(404).json({ error: 'group not found' });
+  res.json(await computeSuggestions(group.id));
+});
```

The route is nested under `/settlements/suggestions`, which reads well
as a sub-resource.

```diff
-    expect(res.body[0].from).toBe(carolId);
-    expect(res.body[0].to).toBe(aliceId);
     expect(res.body[0].amount).toBe(30);
```

This removes `from`/`to` assertions from the debt test, keeping only
`amount`. This weakens the test -- the debt endpoint's value is knowing
*who* owes *whom*, not just how much. These assertions should stay.

```diff
+  test('returns suggestions array', async () => {
+    const res = await request(app).get(`/groups/${groupId}/settlements/suggestions`);
+    expect(res.status).toBe(200);
+    expect(Array.isArray(res.body)).toBe(true);
+    expect(res.body.length).toBeGreaterThanOrEqual(0);
+  });
```

`expect(res.body.length).toBeGreaterThanOrEqual(0)` -- this assertion
is vacuous. Every array has length >= 0. It provides zero confidence.
This assertion should either be removed or replaced with one that
verifies the actual suggestion content (from, to, amount) given the
known test data.

**Primary finding: commit message.** "fix stuff" is not a valid commit
message. This is a feature commit that adds a new endpoint. It should be
`feat: Add settlement suggestion endpoint`. The message provides no
information about what changed or why.

**Secondary finding: test weakening.** Removing `from`/`to` assertions
from the debt test and adding a vacuous assertion to the suggestions
test both reduce confidence. The debt test should retain its directional
assertions. The suggestions test should verify actual content.

**Commit purity:** This commit also modifies existing test assertions
from the previous commit (rounding test formatting, debt test
assertions). Those changes should either be part of commit 2 or
separate.

`[reviewed] ec8cabb, dc5a7d2, 3ab7125 | [pending] 0309b55, 7de1387`

---

### Commit 4: 0309b55 `feat: Add settlement recording`

This commit delivers: a `settlements` table, a `Settlement` model, CRUD
routes for settlements (list + create), tests for the settlements API.
It also rewrites `db.js` formatting and rewrites `groups.test.js`.

```diff
+  db.run(`CREATE TABLE IF NOT EXISTS settlements (id INTEGER PRIMARY KEY AUTOINCREMENT,
+  group_id INTEGER NOT NULL, payer_id INTEGER NOT NULL, payee_id INTEGER NOT NULL,
+  amount REAL NOT NULL CHECK(amount > 0), created_at TEXT DEFAULT (datetime('now')),
+  FOREIGN KEY (group_id) REFERENCES groups(id) ON DELETE CASCADE,
+  FOREIGN KEY (payer_id) REFERENCES members(id),
+  FOREIGN KEY (payee_id) REFERENCES members(id))`);
```

Good: `CHECK(amount > 0)` enforces at the database level. Foreign keys
reference the correct tables.

```diff
+router.post('/groups/:groupId/settlements', async (req, res) => {
+  const group = await Group.findById(req.params.groupId);
+  if (!group) return res.status(404).json({ error: 'group not found' });
+  const { payer_id, payee_id, amount } = req.body;
+  if (!payer_id || !payee_id || !amount) {
+    return res.status(400).json({ error: 'payer_id, payee_id, and amount are required' });
+  }
+  if (payer_id === payee_id) {
+    return res.status(400).json({ error: 'payer and payee must be different' });
+  }
+  if (typeof amount !== 'number' || amount <= 0) {
+    return res.status(400).json({ error: 'amount must be a positive number' });
+  }
+  const settlement = await Settlement.create(group.id, payer_id, payee_id, amount);
+  res.status(201).json(settlement);
+});
```

**Missing validation:** `payer_id` and `payee_id` are not validated as
members of the group. The expense route validates `paid_by` against
group membership (`Member.findById` + `member.group_id !== group.id`).
The settlement route skips this entirely. A client can record a
settlement between members of different groups. This is a real bug --
the foreign key constraint prevents referencing non-existent members,
but it does not enforce group membership.

The `!amount` check will treat `amount: 0` as falsy (caught), but the
`typeof amount !== 'number'` check below is the real gate. The
redundancy is harmless but the `!amount` check is misleading for numeric
fields.

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

Clean model. Follows the same pattern as `Group` and `Expense`.

**Chore mixed in:** The `db.js` diff is massive but almost entirely
whitespace reformatting -- collapsing multi-line SQL and function bodies
into single lines. This is a chore that should be a separate commit.
It makes the actual change (adding the settlements table) hard to spot
in the diff. Same for `groups.test.js` -- the entire test file is
reformatted, removing blank lines and shortening assertions. None of
those changes relate to settlements.

The test reformatting in `groups.test.js` also removes some assertions:
`expect(res.body.name).toBe('Trip to Paris')` becomes just checking
status 201. Same pattern as commit 1 -- weakening existing tests as a
side effect.

**The settlements table DDL is duplicated in 4 places:** `db.js` (the
schema), `balances.test.js`, `expenses.test.js`, `groups.test.js`, and
`settlements.test.js`. Each test file manually creates all tables. This
is a maintenance problem -- adding a column requires changing 5 files.
A shared test helper that initializes the schema would eliminate this
duplication.

**Commit message:** Accurate and under 50 chars. Correct prefix.

`[reviewed] ec8cabb, dc5a7d2, 3ab7125, 0309b55 | [pending] 7de1387`

---

### Commit 5: 7de1387 `feat: Include settlements in balance calculations`

This commit delivers: settlements are now factored into
`computeBalances`, and `computeSuggestions` is rewritten to use
settlement-aware balances instead of delegating to `computeDebts`. A
test verifies that settlements reduce outstanding balances.

```diff
+  const settlements = allRows(db, 'SELECT * FROM settlements WHERE group_id = ?', [groupId]);
...
+  for (const settlement of settlements) {
+    balances[settlement.payer_id] = (balances[settlement.payer_id] || 0) + settlement.amount;
+    balances[settlement.payee_id] = (balances[settlement.payee_id] || 0) - settlement.amount;
+  }
```

The `|| 0` fallback is defensive -- if a settlement references a member
ID that is not in the `balances` map (which was initialized from the
current members list), it still works. But this masks a data integrity
issue: a settlement for a member not in the group should probably be an
error, not silently handled. In practice, given the route validation
(even without group membership checks), the member IDs will be in the
map. The `|| 0` is harmless but suggests the author was uncertain about
the data.

The settlement math is correct: payer's balance goes up (they paid, so
they're owed more / owe less), payee's balance goes down (they received,
so they're owed less / owe more).

```diff
+async function computeSuggestions(groupId) {
+  const balances = await computeBalances(groupId);
+  const debtors = [], creditors = [];
+  for (const [id, bal] of Object.entries(balances)) {
+    if (bal < -0.005) debtors.push({ id: Number(id), amount: -bal });
+    else if (bal > 0.005) creditors.push({ id: Number(id), amount: bal });
+  }
```

This is nearly identical to `computeDebts`. The only differences: (1)
entry thresholds use 0.005 epsilon instead of exact 0, and (2) the
variable name is `suggestions` instead of `debts`. The algorithm body is
copy-pasted.

**DRY violation:** `computeDebts` and `computeSuggestions` are the same
algorithm with different entry thresholds. The fix: make `computeDebts`
also use the 0.005 epsilon (which is the correct behavior anyway -- see
the consistency note in commit 2), then have `computeSuggestions` call
`computeDebts`. Or extract the shared greedy-matching algorithm into a
helper. The current duplication means a bug fix to the matching logic
must be applied in two places.

The conceptual difference is that `computeDebts` shows raw debts from
the current balances, while `computeSuggestions` shows what payments are
still needed. But since `computeBalances` now includes settlements, the
debts endpoint already reflects settlements too. So `computeDebts` and
`computeSuggestions` now return the same data (both call
`computeBalances` which includes settlements). The only difference is
the epsilon threshold. This suggests either `computeDebts` should not
include settlements (showing gross debts) or `computeSuggestions` is
redundant.

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

Good test. A pays 100, split 2 ways = A is owed 50. B settles 30.
A's net balance: 50 - 30 = 20. B's net balance: -50 + 30 = -20.
Assertions match.

**Commit message:** Accurate. Under 50 chars. Correct prefix.

`[reviewed] ec8cabb, dc5a7d2, 3ab7125, 0309b55, 7de1387`

---

## Phase 4: Wrap-up

### Final narrative check

```
7de1387 feat: Include settlements in balance calculations
0309b55 feat: Add settlement recording
3ab7125 fix stuff
dc5a7d2 feat: Add balance and debt endpoints
ec8cabb refactor: Extract expense validation and add balance module
```

The narrative does not read cleanly. "fix stuff" is opaque. "refactor"
is mislabeled. The arc is discernible but requires reading the diffs to
understand.

### Cross-commit consistency pass

1. **Test schema duplication.** Every test file independently creates
   all tables with inline DDL. The settlements table DDL appears in 4
   test files. A shared `createTestDb()` helper would eliminate this.
   Adding a column to any table requires editing every test file.

2. **Assertion weakening pattern.** Commits 1, 3, and 4 all remove
   assertions from pre-existing tests (description value, from/to
   fields, member counts). Each removal individually might be defensible
   as "removing noise," but the cumulative effect is a test suite that
   mostly checks HTTP status codes without verifying response content.
   The creates-an-expense test no longer checks what was created. The
   debt test no longer checks who owes whom. This is a drift toward
   tests that provide little confidence.

3. **Formatting churn.** Commits 1 and 4 both reformat existing code
   (collapsing multi-line SQL, removing blank lines, inlining function
   bodies). This is scattered across feature commits. A single
   upfront chore commit for formatting would keep feature diffs clean.

4. **Validation inconsistency.** The expenses route validates group
   membership of `paid_by`. The settlements route does not validate
   group membership of `payer_id` or `payee_id`. This is a bug.

5. **`computeDebts` vs `computeSuggestions` duplication.** The final
   state of `balances.js` has two nearly identical implementations of
   the greedy debt simplification algorithm.

6. **No error handling on async routes.** None of the route handlers
   wrap their async bodies in try/catch. An unhandled promise rejection
   (e.g., database error) will crash the process or produce a generic
   500, depending on the Express version. This applies to the existing
   codebase too, but the new routes perpetuate it.

### Test and lint status

All 26 tests pass. No linter or type checker configured in the project.

### Summary

**Findings by severity:**

| # | Severity | Finding |
|---|----------|---------|
| 1 | Bug | Settlement route does not validate group membership of payer/payee |
| 2 | Structure | Commit 1 mislabeled as refactor; contains feature work and dead code |
| 3 | Structure | Commit 3 message "fix stuff" is unusable; actually a feature |
| 4 | Design | `computeDebts` and `computeSuggestions` are duplicated algorithms |
| 5 | Quality | Systematic assertion removal weakens test confidence across the branch |
| 6 | Structure | Formatting churn mixed into feature commits obscures real changes |
| 7 | Maintenance | Test schema DDL duplicated in 4 test files |
| 8 | Minor | `computeDebts` entry threshold (exact 0) inconsistent with exit threshold (0.005 epsilon) |

**Recommended commit restructure:**

1. `chore: Reformat test files and db module` (all whitespace/style changes)
2. `feat: Add expense description validation` (validation + its test)
3. `feat: Add balance and debt endpoints` (balance module + routes + tests)
4. `feat: Add settlement suggestion endpoint` (what "fix stuff" actually is)
5. `feat: Add settlement recording` (model, migration, routes, tests)
6. `feat: Include settlements in balance calculations` (wire settlements into balance math)

**Changes made during review:** None. This is a non-interactive review.

**Open items:**
- Settlement route needs group membership validation for payer_id and payee_id
- `computeDebts` and `computeSuggestions` should be deduplicated
- Debt test should restore `from`/`to` assertions
- Suggestions test needs non-vacuous assertions
