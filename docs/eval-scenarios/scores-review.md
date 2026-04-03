# Scorecard: /review Skill Evaluation

## Planted Problem Detection

| # | Problem | Verdict | Explanation |
|---|---------|---------|-------------|
| 1 | Mixed commit (refactor + feature) | **PASS** | Review explicitly flags the atomicity violation in commit 1, identifies it as mixing refactor with feature, and recommends splitting into separate commits with specific structure. |
| 2 | Bad commit message ("fix stuff") | **PASS** | Review identifies the message as vague, flags the wrong prefix (fix vs feat), and proposes the specific replacement `feat: Add settlement suggestion endpoint`. |
| 3 | Missing group membership validation | **PASS** | Review flags that settlement endpoint does not validate payer/payee belong to the group and states the consequence: "A client can record a settlement between members of different groups." |
| 4 | Missing "all settled" edge case test | **FAIL** | Review notes general test weakening and vacuous assertions in the suggestions test, but never identifies the specific missing edge case of "all debts already settled" from the spec. |
| 5 | Zero-amount suggestions vs empty array | **FAIL** | Review does not identify that the implementation returns zero-amount suggestions instead of an empty array when all debts are settled; the spec divergence goes undetected. |
| 6 | Division by zero on empty group | **FAIL** | Review discusses `members.length` as the split denominator and notes the retroactive membership concern, but never flags the zero-member division-by-zero risk or the spec's empty-group edge case. |

## Review Quality

| Criterion | Verdict | Explanation |
|-----------|---------|-------------|
| Suggests specific fixes | **PASS** | Each identified problem includes a concrete recommended action (split commits, reword message, add validation code). |
| Verifies against spec | **FAIL** | Review does not cross-reference the behavior inventory or edge case list from the spec; problems 4-6 were missed because no spec verification was performed. |
| No false findings | **PASS** | Extra findings (DRY violation, assertion weakening, test schema duplication, epsilon inconsistency) are all legitimate observations, not manufactured problems. |
| Reviews commit by commit | **PASS** | Review walks through all 5 commits sequentially with dedicated sections evaluating each on its own terms. |

## Score Summary

| Category | Items | Score |
|----------|-------|-------|
| Planted problems (1-6) | 3 PASS, 0 PARTIAL, 3 FAIL | 3.0 / 6.0 |
| Review quality (7) | 3 PASS, 0 PARTIAL, 1 FAIL | 3.0 / 4.0 |
| **Overall** | **6 PASS, 0 PARTIAL, 4 FAIL** | **6.0 / 10.0** |

## Notes

The review excelled at structural and process-level findings (atomicity, commit messages, missing validation) but missed all three problems in commit 5 that required checking the implementation against the spec's edge case inventory. The review had no spec to cross-reference, which likely explains the gap -- it analyzed the code on its own terms rather than against defined requirements. The commit-by-commit format and specificity of recommendations were strong.
