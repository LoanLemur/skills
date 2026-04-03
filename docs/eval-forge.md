# Forge Skill Evaluation

Evaluated against: `forge/SKILL.md`, `docs/24-skill-writing-craft.md`, `docs/28-skill-map.md` (forge entry), `docs/19-ralph-loop.md`, `docs/00-synthesis.md`, `docs/08-agent-skills-standard.md`.

---

## 1. Broken — Agent would produce wrong output or get stuck

### 1.1 No iteration ceiling or stuck-detection threshold

`docs/19-ralph-loop.md` section 6 is explicit: "All implementations cap the maximum number of loops" — without caps the failure mode is expensive loops burning API credits. The skill defines STUCK as "Multiple iteration cycles haven't resolved the issues" but "multiple" is not a number. An agent has no threshold for when to stop iterating and surface. It may run 10 cycles or 2 before deciding it's stuck, or loop indefinitely.

Fix: Add a specific ceiling to the Rules section — "After 3 full Phase 3→5 cycles without resolution, declare STUCK and surface to the user with the full iteration history."

### 1.2 Test C conditionality creates a silent skip

Phase 3 Test C says "for enhanced skills only" and "If an existing version of the skill exists." The condition is never mechanically defined. The Input section says to check `~/.claude/skills/{name}/SKILL.md` as a starting point, but Phase 3 doesn't reference that check. An agent that drafted a new skill from scratch can conclude "no existing version" and skip Test C without ever checking the filesystem — especially since the path check happens at the start of the session and may not be in working memory by Phase 3.

Fix: "Check whether `~/.claude/skills/{name}/SKILL.md` exists. If it does, Test C is required. Record this finding in Phase 1 so Phase 3 can reference it."

### 1.3 Phase 6 writes files without specifying the target path

Step 3 of Phase 6: "On approval, write the files to the target location." There is no definition of "the target location" anywhere in the skill. The Input section references reading from `~/.claude/skills/{name}/SKILL.md` but never says to write there. An agent could write to the current working directory, a subdirectory of the skills repo, or prompt the user — all wrong.

Fix: "Write to `~/.claude/skills/{name}/`, creating the directory if it does not exist."

### 1.4 Adversarial test prompt contains persona framing the skill itself prohibits

Phase 3, Test A instructs the sub-agent with: "You are testing this skill for loopholes." Phase 2 anti-patterns explicitly prohibits persona framing ("You are a..."), citing the PRISM paper. The forge skill violates the exact rule it enforces in every skill it builds. An adversarial agent reading this would correctly flag it as a loophole in forge itself.

Fix: Replace with imperative framing: "Read these instructions and find every way an agent could: skip steps, produce shallow output, declare done prematurely, avoid the hard parts, generate verbose filler, or follow the letter while violating the spirit. Report each loophole found. If the skill is tight, say so."

---

## 2. Unclear — Agent might interpret this multiple ways, some wrong

### 2.1 Phase 1 gate contradicts the autonomous operation model

Phase 1 ends: "State back what you understand...Do not proceed without this." The skill's header (line 15) says "Only surface to the user when all tests pass or you're stuck," implying fully autonomous operation. These two instructions conflict. In a conversational session, "do not proceed without this" means wait for user acknowledgment. In an autonomous loop, there's no one to acknowledge. An agent that takes the gate literally will pause at Phase 1 and wait; an agent that takes the header literally will skip or ignore the gate.

Fix: Clarify: "State your understanding internally and proceed. Do not surface to the user at this stage."

### 2.2 PASS condition is unsatisfiable for new skills as written

Phase 4: "PASS — All three tests report no issues." For a new skill, Test C does not run and therefore reports nothing — it doesn't "report no issues," it doesn't report at all. The condition as written is never satisfied for new skills. An agent reading strictly will be stuck in an undefined state: it cannot reach PASS because Test C has no output.

Fix: "PASS — All applicable tests report no issues. (For new skills: Tests A and B. For enhanced skills: Tests A, B, and C.) Proceed to Phase 6."

### 2.3 "Relevant craft analysis docs (docs/20-25)" is underspecified

The Input section says "The relevant craft analysis docs (docs/20-25) for patterns." This is six documents. No guidance is given on which are relevant to which skill type, whether all six should be read every time, or how to determine relevance. The synthesis (`docs/00-synthesis.md` Part 11) provides a ranked reading order with MUST READ labels, but the forge skill doesn't reference it. An agent reading forge in isolation will not know whether to read all six or a subset.

Fix: Either list the always-required documents explicitly, or say "Read in the order specified in `docs/00-synthesis.md` Part 11, prioritizing docs marked MUST READ."

### 2.4 Phase 4 PASS category omits the next step

Every other Phase 4 category specifies what to do next ("Proceed to Phase 5," "Surface to the user"). PASS does not. The logical next step is Phase 6 but the agent must infer it. Inference is appropriate here but inconsistency in the pattern creates a small ambiguity.

Fix: "PASS — All applicable tests report no issues. Proceed to Phase 6."

---

## 3. Missing — Scenarios the skill doesn't handle

### 3.1 No failure mode for missing skill-map entry

If the argument is a skill name with no entry in `docs/28-skill-map.md`, Phase 1 step 1 ("Read the skill map entry") fails silently. There is no "stop and report" for this case. The skill-map currently has five entries. Any other skill name would leave the agent without a spec, causing it to either fabricate one or loop confused.

Fix: Add to the failure modes or Rules: "If the skill name has no entry in `docs/28-skill-map.md`, stop immediately. Ask the user to provide the spec, acceptance criteria, and goal-level description directly before proceeding."

### 3.2 No guard against self-application (/forge forging /forge)

The skill-map notes: "/forge — Built first, by hand (there is no forge to forge the forge)." The skill itself has no guard. Passing `forge` as the argument would cause the agent to attempt to build a new version of the skill it is currently executing from. This is either benign (it just drafts a new forge skill) or produces a confusing recursive situation. Either way it needs a guard.

Fix: Add to the Input section or Rules: "If the argument is `forge`, stop. Forge cannot forge itself. Surface this to the user."

### 3.3 No output format constraints for Phase 6 delivery

Acceptance criterion 5 requires every skill forge builds to have "explicit output format with constraints." Forge itself has none for its Phase 6 delivery. Phase 6 lists four items to present but gives no constraints on format, length, or depth. The iteration history could be a one-liner or five paragraphs. The test results summary has no template. This is the most direct meta-inconsistency in the skill.

Fix: Add an output format block. Minimum: "Iteration history: one line per cycle (what changed and why). Test results: PASS/FAIL for each test with one sentence of evidence. Skill content: inline only if under 100 lines, otherwise reference the written file path."

### 3.4 No fallback when sub-agent spawning is unavailable

The skill's entire testing mechanism depends on spawning sub-agents. No guidance is given for environments where the Agent tool is unavailable or restricted. The synthesis (P9) explicitly notes skills should be teachable to mid-level engineers in varied environments.

Fix: Add to the Rules: "If sub-agent spawning is unavailable, surface to the user immediately. Do not evaluate the skill yourself — self-evaluation defeats the purpose of fresh context."

---

## 4. Convention Violations — Don't cause failure but contradict craft standards

### 4.1 Description partially summarizes the workflow

The description opens: "Build, test, and iterate on agentic skills using a Ralph-style loop." Per `docs/24-skill-writing-craft.md`: "Descriptions must contain only triggering conditions." "Build, test, and iterate...using a Ralph-style loop" describes mechanism, not trigger. The craft doc warns that process summaries in descriptions cause agents to follow the summary and skip the body. Forge's description is closer to compliant than most offenders (the rest is triggering conditions), but the first clause is a violation.

The description is 51 words — passes the 50-word floor but barely.

Fix: Rewrite the opening clause as a trigger: "Use when creating or enhancing an agentic skill." Move mechanism to the body.

### 4.2 Body opener repeats a full workflow summary

Lines 14-16:
> "Build or enhance a skill through automated iteration. Each cycle: draft → test → evaluate → improve. Fresh context per test. Only surface to the user when all tests pass or you're stuck."

This is a complete process summary at the top of the body. Combined with the description, an agent has now seen the full workflow twice before reading Phase 1. The craft doc warns this is the failure mode — agents follow summaries and skim detail. The opener should be one sentence stating what forge does, not a synopsis of all six phases.

Fix: "Forge produces a tested SKILL.md (and supporting files) for the skill named in $ARGUMENTS."

### 4.3 `docs/08-agent-skills-standard.md` is missing from the reading list

`docs/00-synthesis.md` Part 11 marks `08-agent-skills-standard.md` as "MUST READ" and ranks it second in the reading order. The forge Input section lists four docs to read but omits doc 08. Phase 2 references it by name ("YAML frontmatter: name, description..."), implying the agent should know the spec — but it's not instructed to read it.

Fix: Add `docs/08-agent-skills-standard.md` to the "Also read" list.

### 4.4 Motivational framing in the Rules section

Rules section: "The iteration loop is the point — more cycles is expected, not failure."

This is reassurance aimed at the agent — equivalent to sycophancy directed inward rather than outward. CLAUDE.md prohibits hedging and inflated framing. This line doesn't change behavior; it's noise.

Fix: Remove it.

---

## 5. Nitpicks

### 5.1 "Also read" list has no prioritization

Four items are listed flat with no signal about which are required vs. optional. The catch-all "The relevant craft analysis docs (docs/20-25)" is actually six large documents. If read eagerly on every invocation, the context budget is consumed before Phase 1 is complete. Progressive disclosure — the skill's own core principle — is not applied to forge's own reading list.

### 5.2 Acceptance criteria belong in a reference file

The Acceptance Criteria section (lines 174-208) serves two purposes: guidance for the forge agent itself, and content to pass to Test B sub-agents. Its placement inside SKILL.md means Test B sub-agents must receive the entire forge SKILL.md to get the criteria — or the forge agent must copy-paste the section into the sub-agent prompt. Extracting to `references/acceptance-criteria.md` would allow Test B to receive a clean, minimal context without all of forge's implementation details.

### 5.3 The "Rules" section duplicates instructions already in the phases

Rules include: "Never weaken tests to make a skill pass" (also in Phase 5), "Never declare a skill done without running all three tests" (also in Phase 3), "Fresh context for every test" (also in Phase 3). Duplication dilutes instruction weight and wastes the context budget. Per craft convention: if removing an instruction wouldn't cause a mistake because it's stated elsewhere, remove the duplicate.

---

## Goal-Level Check

The skill-map entry for /forge: "Build, test, and iterate on skills using a Ralph-style loop. Output: A tested skill that passes adversarial, two-Claude, and comparison tests."

The skill achieves this goal if all the "Broken" issues above are fixed. The structure (draft → test → evaluate → improve → deliver) is correct and faithful to the Ralph loop. The three-test pattern (adversarial, two-Claude, comparison) maps directly to the skill-map's output requirement. The failure modes that matter are 1.1 (no iteration ceiling), 1.2 (silent Test C skip), 1.3 (unknown write target), and 1.4 (persona framing the skill prohibits). Without these fixes, a faithful agent either loops indefinitely, skips required tests, writes files to an unknown location, or produces skills that violate the standard forge enforces.
