# Scores: /review v2

Evaluated against planted problems in `scenario-review.md`.

---

## 1. Catches Mixed Commit (Problem 1)

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Identifies the atomicity violation | PASS | Flags commit 1 as mixing refactor (expense validation) with feature (balance module). States "Two concerns in one commit" and notes the balance module is dead code at this point in history. Identifies the category violation explicitly. |
| Recommends a specific fix | PASS | "Split this commit: refactor (validation + test cleanup) in one, balance module in another (or fold into commit 2)." Concrete two-commit split recommendation. |

**Category: PASS**

---

## 2. Catches Bad Commit Message (Problem 2)

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Flags "fix stuff" | PASS | "fix stuff is the worst possible commit message. Wrong prefix (fix: implies a bug fix), no description of what changed, violates every message convention. The actual content is a feature addition." Identifies vagueness, wrong prefix, and non-descriptive nature. |
| Suggests a replacement | PARTIAL | Recommends "Rename or drop this commit entirely. The suggestions feature should be a single commit after settlement recording, with the real implementation from commit 5." This is a structural recommendation rather than a specific replacement message string like `feat: Add settlement suggestion endpoint`. |

**Category: PARTIAL** (one PASS, one PARTIAL)

---

## 3. Catches Missing Validation (Problem 3)

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Identifies missing group membership check | PASS | "No check that payer_id and payee_id are valid members of this group." Explicitly flagged in commit 4 review. |
| Explains the consequence | PASS | "A client could post arbitrary member IDs from another group. The foreign key constraint would catch cross-group references only if the member ID does not exist at all -- it would not catch a valid member from a different group." Clear consequence with nuance about FK limitations. |

**Category: PASS**

---

## 4. Catches Missing Edge Case Test (Problem 4)

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Identifies the missing "all settled" test | PARTIAL | The review does not explicitly call out the missing "all debts already settled" test case. It notes "empty after filter: If all balances are within the 0.005 threshold (everyone nearly settled), both arrays are empty, while loop does not execute, returns empty suggestions. Correct." -- this is an edge case audit of the code path but does not flag the absence of a test covering it. No explicit finding says "add a test for the all-settled scenario." |

**Category: PARTIAL**

---

## 5. Catches Spec Divergence (Problem 5)

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Identifies zero-amount suggestions issue | PASS | "computeSuggestions can produce zero-amount suggestions. Unlike computeDebts which has if (rounded > 0), computeSuggestions does not guard against this. This is a bug -- zero-amount suggestions can appear in the output." Clearly identified. |
| References the spec | FAIL | The review identifies this as a bug by comparing computeSuggestions to computeDebts, but never references the spec's requirement that suggestions should return an empty array when all settled. The finding is framed as internal inconsistency, not spec divergence. |

**Category: PARTIAL** (one PASS, one FAIL)

---

## 6. Catches Division-by-Zero Bug (Problem 6)

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Identifies the zero-member risk | PASS | "If a group has zero members, this divides by zero producing Infinity. A group with expenses but no members is unlikely but not impossible (members could be deleted). This should guard against empty members or return early." Explicitly flagged in commit 1 review. |
| References the spec | FAIL | Does not reference the spec's listing of "empty group" as an edge case requiring an empty array response. The finding is framed as a general defensive coding concern. |

**Category: PARTIAL** (one PASS, one FAIL)

---

## 7. Quality of Review Process

| Criterion | Score | Evidence |
|-----------|-------|----------|
| Suggests specific fixes | PASS | Each finding includes a concrete recommendation: split commits, add the `> 0` guard, add member-existence checks, return early on empty members. |
| Verifies against spec | FAIL | The review never cross-references a spec or behavior inventory. All findings are derived from code analysis and internal consistency checks. No mention of a design spec or edge case list from a spec document. |
| Does not manufacture false findings | PASS | Commit 2 is assessed as clean ("No defects found"). The additional findings (DRY concern, test assertion drift, ordering) are legitimate observations, not false positives. No unnecessary work recommended. |
| Reviews commit by commit | PASS | Full sequential commit-by-commit walkthrough with dedicated sections for each of the 5 commits. |

**Category: PARTIAL** (three PASS, one FAIL)

---

## Category Totals

| Problem | Category | Score |
|---------|----------|-------|
| 1. Mixed commit (atomicity) | Atomicity | PASS |
| 2. Bad commit message | Message quality | PARTIAL |
| 3. Missing group membership validation | Correctness | PASS |
| 4. Missing "all settled" test | Test coverage | PARTIAL |
| 5. Zero-amount suggestions vs empty array | Spec divergence | PARTIAL |
| 6. Division by zero on empty group | Correctness | PARTIAL |
| 7. Review process quality | Process | PARTIAL |

---

## Overall Score

- **PASS:** 2 / 7
- **PARTIAL:** 5 / 7
- **FAIL:** 0 / 7

**Weighted score (PASS=1, PARTIAL=0.5, FAIL=0): 4.5 / 7 = 64%**

### Summary

The review correctly identified all 6 planted problems at some level -- none were missed entirely. Strengths: atomicity analysis, missing validation catch, and the zero-amount suggestion bug were all well-articulated with concrete fixes. The main weakness is that the review never references the spec document, so findings that should be framed as spec divergences are instead framed as internal code inconsistencies or general defensive programming. The missing "all settled" test was noted as an analyzed code path but not flagged as a gap in test coverage. The commit message critique was thorough on diagnosis but stopped short of proposing a specific replacement message.
