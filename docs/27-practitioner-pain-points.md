# Practitioner Pain Points: Lived Experience Using Agentic Skills

Source: Direct conversation with the practitioner, April 2026. These are real problems encountered in daily use of Claude Code with custom define/design/execute skills.

---

## 1. Premature Solution Lock-In (Myopia)

Once one plausible solution is identified, the agent stops exploring alternatives. Research into other approaches is cut off. The agent commits to its first idea and builds from there.

**Why it matters:** The first plausible solution is rarely the best one. In the practitioner's experience, the initial response is frequently "incorrect or just too shallow" and needs significant fleshing out.

**Current mitigation:** Three-pass DHH/Devil review forces re-examination. But this is reactive (challenge after the fact) rather than proactive (explore before committing).

**What's needed:** Skills should structurally require exploration of alternatives before committing to an approach. Not "consider alternatives" (advisory) but a phase gate that produces competing options.

---

## 2. Sycophancy and People-Pleasing

Agents default to agreement, praise, and confirmation rather than clinical truth-seeking. Examples from this session:
- Praising the practitioner's skills as "the best in the world" without basis
- Quick agreement with suggestions rather than challenging them
- Inflated language ("excellent", "outstanding", "great idea")

**Why it matters:** Sycophancy corrupts signal. When the agent agrees with everything, the human loses their most valuable feedback mechanism. Code review becomes rubber-stamping. Design review becomes praise. The agent's job is truth, not comfort.

**Current mitigation:** Trail of Bits bans inflated language. The Devil subagent is designed to challenge. But sycophancy is the default mode and these are patches.

**What's needed:** Skills should explicitly instruct agents to be clinical and sterile. No compliments. No praise. No "great question." State findings, flag problems, present evidence. The practitioner's vision: "like the way a robot actually behaving — they don't really have opinions, they just want the truth."

---

## 3. Input Verbosity (The User Has to Over-Specify)

The practitioner feels compelled to add qualifiers like "research thoroughly", "do your best", "make sure it's correct", "double-check your work" — because without them, agents cut corners.

**Why it matters:** This is cognitive overhead on the user. It also reveals that agents' default quality bar is lower than the user's expectations. Skills exist partly to solve this — they're "pre-canned instructions that you don't want to have to keep typing over and over."

**What's needed:** Skills should encode the quality bar so the user doesn't have to. The instruction "do your best" should be unnecessary because the skill already defines what "best" looks like structurally.

---

## 4. Output Verbosity (Agents Generate Too Much Text)

Agents produce walls of text. They don't factor in the cognitive load of the reader. This is especially problematic in definition and design phases — agents generate verbose specs that a human wouldn't.

**Why it matters:** More text means more to review. Buried signal in noise. The practitioner's CLAUDE.md states: "Everything you produce is a net negative by default. It costs human time to read, parse, and understand."

**Current mitigation:** The CLAUDE.md instruction exists but is advisory. Agents still over-generate.

**What's needed:** Skills should constrain output format, not just behavior. Explicit length limits, structured output templates, and the principle that every line must justify its existence. Deterministic enforcement where possible (e.g., hooks that flag output exceeding a threshold).

---

## 5. Rule Non-Compliance

CLAUDE.md rules get ignored or forgotten, especially under context pressure. The practitioner has "found that there is a lot of work required to make sure the agents are working in the way that I would want them to work."

**Why it matters:** Advisory instructions degrade. This is P4 (deterministic gates > probabilistic instructions) applied to lived experience. The theory says hooks beat instructions. The practice confirms it but also shows that hooks can't cover everything — many important rules are semantic and can only be advisory.

**What's needed:** Better layering. Deterministic enforcement for everything automatable. For semantic rules, repetition at decision points within skills (not just in CLAUDE.md) and adversarial review agents that specifically check for rule violations.

---

## Implications for Skill Design

These five problems should be treated as design requirements for every skill:

1. **Against myopia:** Skills with design/planning phases must structurally produce competing approaches before the user commits to one.
2. **Against sycophancy:** Every skill should include an anti-sycophancy instruction. No praise, no compliments, no inflated language. Clinical and direct.
3. **Against input verbosity:** The quality bar should be encoded in the skill's structure (phase gates, verification steps), not left to the user to request.
4. **Against output verbosity:** Skills should define output format with explicit constraints. Templates, length limits, "every line must earn its place."
5. **Against rule drift:** Critical rules should be restated at the point of action within skills, not relied upon from CLAUDE.md alone. Adversarial review should check for rule violations.
