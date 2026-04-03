# /execute Skill Evaluation

---

## 1. Broken — Agent would produce wrong output or get stuck

### B1. Checklist paths are unresolvable

Lines 136-138 and 208-209 reference `checklists/convention-review.md` and `checklists/adversarial-review.md` with no anchor path. When a sub-agent is spawned, its working directory is not defined in the skill. A fresh sub-agent reading "Load `checklists/convention-review.md`" has no way to know this path is relative to the skills repo root (`/Users/nick/repos/skills/checklists/`), not the project being implemented. The agent will either guess wrong or fail silently. Use an absolute path or an explicit `@`-import reference.

### B2. Sub-agent review structure is contradictory

Line 117: "Spawn two sub-agent reviews. Each gets fresh context." Lines 120-127: "Provide each reviewer with: [same list for both]." Then lines 136-138 define Pass 1 (convention) and Pass 2 (adversarial) as if they differentiate the two sub-agents. But "Provide each reviewer with" + the structural prompt instruction "include in both reviews" implies both agents get the same inputs before the pass differentiation. An agent reading this will either:

- Give both checklists to both sub-agents (wrong — defeats the two-pass model)
- Give the structural prompt to only one (wrong — "include in both" says otherwise)
- Correctly spawn one agent with the convention checklist and one with the adversarial checklist, but this requires inferring that "Pass 1" and "Pass 2" map to the "two sub-agent reviews" from line 117 — an inference the text doesn't make explicit

Fix: Restructure as "Spawn two sub-agents. Sub-agent 1 (convention): [inputs + Pass 1 checklist]. Sub-agent 2 (adversarial): [inputs + Pass 2 checklist]. Both receive the structural prompt."

### B3. Phase 2 branch naming has no resolution path

Line 60: "Use the branch name the user specifies, or ask." The argument to the skill is a GitHub issue URL or number, not a branch name. Nothing in Phase 1 or Phase 2 captures a branch name from the user before this step. "The user specifies" refers to nothing — the user didn't specify a branch name in the invocation. An agent following this literally will ask the user for a branch name every time, even when a sensible default (derived from the issue title/number) is available. Fix: "Derive from the issue number and title (e.g., `issue-123-add-payment-flow`), or ask if the user has a convention."

### B4. "Compact the conversation" instruction is incomplete

Line 179: "After every 3-4 commits, or when the session has been running extensively, compact the conversation." The synthesis doc (P7) distinguishes `/compact` (compresses context, preserves it) from `/clear` (wipes context). Line 181 also mentions `/clear` as an option. An agent receiving "compact the conversation" without a specific command may not know which operation to use, or may use `/clear` when `/compact` was intended and lose critical context mid-implementation. Fix: Specify the command explicitly — "run `/compact` to summarize context" vs. "run `/clear` and restart the current commit" — and state when each applies.

---

## 2. Unclear — Agent might interpret multiple ways, some of them wrong

### U1. "Relevant specs" in Phase 3b is undefined

Line 111: "Run relevant specs." An agent implementing a large feature has no definition of "relevant." Does this mean only specs for files touched in this commit? The entire test suite? The model or controller spec for the resource being built? Without a decision rule, the agent will make its own judgment — sometimes running only the new spec (missing regressions) and sometimes running the full suite (taking minutes for a small change). Fix: "Run specs for files touched in this commit plus any specs that exercise the code paths those files affect. If unsure, run the full suite."

### U2. "Commits 4+" is ambiguous

Line 124: "For commits 4+, also provide the cumulative diff." Does "commits 4+" mean the 4th commit on the branch (i.e., once there are 3 prior commits to compare against), or the 4th iteration through the Phase 3 loop? For a single-pass read this is probably understood as "the 4th commit onward," but the phrasing "commits 4+" is unusual enough that an agent might misread it as "for reviews of 4 or more commits at once." Fix: "Starting from the 4th commit on this branch, also provide..."

### U3. Does splitting a capability count as a deviation requiring an issue comment?

Lines 91-93: "You may discover a planned capability should be two or three commits... Split without asking. Comment on the issue to document the deviation." Phase 3e then says: "If a deviation was needed, comment on the GitHub issue explaining what changed and why." Both passages point to the same action (issue comment), but the Phase 3 loop instructs the comment during the split decision, while Phase 3e instructs it after committing. An agent might comment twice (once when deciding to split, once at 3e), or might interpret 3e as the authoritative step and delay the comment. Fix: Collapse the two mentions into one: "Splits are permitted without asking. Document the split with an issue comment at the time of the first sub-commit (not again at 3e)."

### U4. "Re-run verification before committing" is ambiguous after review feedback

Line 154: "If you make changes based on feedback, re-run verification before committing." Phase 3 is structured as 3a → 3b → 3c → 3d. "Re-run verification" implies going back to 3b, but 3b's instruction says "before committing, verify" — which is already satisfied before 3c ran. Does "re-run verification" mean re-execute 3b in full, or just re-run the test suite? Fix: "Re-run the test suite (3b) after applying review feedback. If a migration changed, also re-verify it runs cleanly."

### U5. Phase 1 behavior inventory output format is unspecified

Lines 44-52: "Extract every distinct behavior... If not, construct one from the design narrative. Present a summary... The behavior inventory (numbered)." The skill says to number the inventory but gives no format guidance — should it match a specific template, use checkboxes, include the capability mapping inline? An agent will invent a format, and if /review later cross-references this inventory, format inconsistency may cause the cross-reference to fail. Fix: Provide a minimal example or format spec for the behavior inventory output.

---

## 3. Missing — Scenarios the skill doesn't handle

### M1. No stop condition when the issue has no capabilities list

The description says "DO NOT use when the issue lacks an implementation plan," but Phase 1 has no explicit stop-and-report path for this case. An agent that encounters an issue with a design narrative but no ordered capability list will proceed through Phase 1's completeness check and may construct its own capability list rather than stopping. The skill-map's failure mode for /execute is: "The design won't work in practice. Here's why: [specific problems]. Return to /design." This failure mode appears in the Failure Modes section at the bottom but is not integrated as an early-exit in Phase 1. Fix: Add to Phase 1 step 2: "If no ordered capability list exists, stop. Report: 'This issue has no implementation plan. Run /design first.'"

### M2. No guidance for merge conflicts

The skill creates a branch off main (Phase 2) and works on it across multiple commits. If main advances while the branch is being built, the agent will eventually need to rebase or merge. The skill has no instruction for this — an agent encountering a conflict mid-implementation has no guidance on whether to rebase, merge, or ask the user. Fix: Add to Failure Modes or Phase 2: "If main has advanced since branching, rebase before continuing: `git rebase main`. On conflict, stop and ask the user."

### M3. No guidance on what context sub-agents receive beyond the diff

Phase 3c specifies what to pass to reviewers (planned commit description, capability, the diffs, structural prompt, checklist). The synthesis doc (P5: Selective Context Transfer) says pass the spec and review checklist, strip conversation history. The skill omits passing the issue body/design to reviewers. A sub-agent with only a diff and a commit description cannot check whether the implementation matches the design intent — it can only check code quality. Fix: Add "the relevant section of the issue's design narrative" to the reviewer's context inputs.

### M4. No stop condition for Phase 2 when the branch already exists or main has uncommitted changes

Phase 2 runs `git checkout main && git pull && git checkout -b {branch-name}`. If the branch already exists (e.g., resuming after a `/clear`), the command fails. If the working tree is dirty, `git checkout main` fails. The skill has no instruction for either case. Fix: Add a precondition check: "Verify `git status` is clean before branching. If the branch already exists, check it out rather than creating it: `git checkout {branch-name}`."

### M5. No guidance on the /review handoff

The skill-map shows /execute feeding into /review. Phase 5 (Report) ends with "Do NOT push or create a PR unless the user asks" but gives no instruction about invoking /review or what to pass to it. A user following the pipeline would naturally ask "what next?" and the skill is silent. This isn't a workflow step the skill must execute, but a handoff note ("Next: run /review on this branch with the issue number") would close the pipeline loop. The skill-map shows /execute → /review as explicit. Fix: Add to Phase 5: "When ready for final review, run `/review` with the issue number and branch."

---

## 4. Convention Violations — Don't follow craft conventions but won't cause failure

### CV1. Introductory paragraph below H1 summarizes the workflow

Lines 16-18:
```
Implement a feature end-to-end from a GitHub issue. Work commit by
commit, review each commit independently, and maintain clean git history.
```

The craft doc (Part 10, "The Description Field Is Make-or-Break") states: "Descriptions that summarize the workflow cause Claude to follow the summary and skip the skill body entirely." While this text is below the H1 (not in the frontmatter description), it's the first thing an agent reads after activation. It's a workflow summary. It wastes tokens and risks the agent treating it as authoritative over the phases. Cut it or make it a one-line purpose statement that doesn't describe process.

### CV2. Rules section duplicates phase content

Lines 250-259 (Rules section): Every rule is already enforced structurally by the phases. "One commit at a time" is enforced by Phase 3's loop structure. "Review every commit" is enforced by Phase 3c. "Clean history matters" restates the Handling Mistakes section. "Comments over edits" restates Phase 3e. Duplication wastes instruction budget and, per the craft doc, dilutes compliance by adding redundant instructions. The only rule adding genuine new constraint is "Follow the plan, but trust the design" (the capability list vs. design source-of-truth distinction). Cut the rest or fold them into the phases where they apply.

### CV3. "Do not perform reviews inline" appears twice, identically

Line 140 and line 211. One occurrence is inside Phase 3c; the other inside Phase 4's final review. Identical phrasing serves no purpose at the second occurrence since the instruction is already internalized. Cut the second.

### CV4. No explicit output format with constraints for Phase 5

The synthesis doc and craft conventions require an explicit output format. Phase 5 says "Present a summary to the user" with four bullet points of content. It specifies no format constraints (length, structure, heading style). The skill-map says the output of /execute is "Commits on a feature branch, deviations as issue comments" — this is the real output. Phase 5's summary is a report, not the output. Either specify the report format explicitly or acknowledge it's freeform.

### CV5. "Ask when stuck" in Rules is vague

Line 259: "Ask when stuck. If something doesn't work in practice, ask the user." The craft doc warns against vague instructions ("be careful with the database" vs. "never run DROP without approval"). "Ask when stuck" is vague — stuck how? After how many attempts? The Failure Modes section already quantifies one case ("After 3-5 attempts on the same failure, stop"). The Rules entry should either point to the Failure Modes section or be cut.

---

## 5. Nitpicks

### N1. "Commits 4+" phrasing is unusual

"For commits 4+, also provide the cumulative diff" (line 124). The `4+` shorthand reads as code, not prose. "Starting from the fourth commit" is clearer and consistent with the skill's prose style.

### N2. Phase 3e heading "Check Alignment" is not literal

The actual content of 3e is about documenting deviations. "Document Deviations" would match the imperative style used elsewhere and make the phase scannable without reading the body.

### N3. HEREDOC mention lacks an example

Line 159: "Use HEREDOC for the message." An agent that doesn't know the exact HEREDOC syntax for `git commit` may use the wrong form. A one-line example (`git commit -F- <<'EOF'`) would prevent this — consistent with the craft doc's "one snippet beats three paragraphs."

### N4. Phase 1 step 3 and step 4 both say "if ... stop" but have different responses

Step 3 (line 37): "ask the user before proceeding." Step 4 (line 47): "stop. Ask the user whether to add it, defer it, or confirm." The distinction is meaningful (step 3 is a blocking question about outstanding decisions; step 4 is a gap-in-the-plan question with three options). The presentation looks parallel but the responses are different. Naming the two stop conditions differently ("blocking question" vs. "plan gap") would reduce agent confusion about which pattern to apply.

### N5. Line count

The skill is 260 lines. The craft doc sets the limit at 500 lines. Well within budget, leaving room for the fixes above without restructuring.

---

## Summary Table

| ID | Severity | Location | One-line description |
|----|----------|----------|----------------------|
| B1 | Broken | Lines 136-138, 208-209 | Checklist paths are relative with no anchor — unresolvable in sub-agent context |
| B2 | Broken | Lines 117-140 | Sub-agent review structure contradicts itself on what each agent receives |
| B3 | Broken | Line 60 | Branch name resolution has no fallback — "user specifies" refers to nothing |
| B4 | Broken | Line 179 | "Compact the conversation" doesn't specify the command or when to use /clear vs /compact |
| U1 | Unclear | Line 111 | "Relevant specs" is undefined — agent will guess scope |
| U2 | Unclear | Line 124 | "Commits 4+" is ambiguous — 4th commit on branch vs. 4th loop iteration |
| U3 | Unclear | Lines 91-93, 163-169 | Split-capability deviation: two instructions for the same issue comment |
| U4 | Unclear | Line 154 | "Re-run verification" after review doesn't specify scope |
| U5 | Unclear | Lines 44-52 | Behavior inventory format unspecified — /review cross-reference may fail |
| M1 | Missing | Phase 1 | No stop condition when issue has no capabilities list |
| M2 | Missing | — | No guidance on merge conflicts if main advances during implementation |
| M3 | Missing | Phase 3c | Sub-agents don't receive the design narrative — can't check design intent |
| M4 | Missing | Phase 2 | No precondition check for dirty working tree or existing branch |
| M5 | Missing | Phase 5 | No /review handoff instruction to close the pipeline loop |
| CV1 | Convention | Lines 16-18 | Introductory paragraph is a workflow summary — wastes tokens, risks shortcircuit |
| CV2 | Convention | Lines 250-259 | Rules section duplicates phase content — dilutes instruction budget |
| CV3 | Convention | Lines 140, 211 | "Do not perform reviews inline" appears twice identically |
| CV4 | Convention | Phase 5 | No explicit output format or constraints for the Phase 5 report |
| CV5 | Convention | Line 259 | "Ask when stuck" is vague — already quantified in Failure Modes |
| N1 | Nitpick | Line 124 | "Commits 4+" — use "Starting from the fourth commit" |
| N2 | Nitpick | Phase 3e | Heading "Check Alignment" — rename to "Document Deviations" |
| N3 | Nitpick | Line 159 | HEREDOC reference lacks a one-line syntax example |
| N4 | Nitpick | Lines 37, 47 | Two "stop" conditions look parallel but have different responses — name them differently |
| N5 | Nitpick | — | 260 lines — well under 500-line limit, room for fixes |
