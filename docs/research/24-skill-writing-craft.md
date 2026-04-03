# The Craft of Writing Agent Skills

**Purpose:** Practical guide to writing instructions that AI agents actually follow. The meta-skill — how to write skills well.

**Sources:** Anthropic official docs (code.claude.com/docs/en/skills, platform.claude.com skill authoring best practices), GitHub blog analysis of 2,500+ agents.md files (Matt Nigh), Addy Osmani "How to Write a Good Spec for AI Agents," Martin Fowler "Context Engineering for Coding Agents," Stack Overflow "Building Shared Coding Guidelines for AI," HumanLayer "Writing a Good CLAUDE.md," Claude Code source leak analysis (Piebald-AI), Cursor best practices documentation.

---

## Part 1: Why Agent Instructions Are Different From Human Instructions

Humans read instructions with enormous tacit context. An engineer reading "follow REST conventions" knows what REST conventions are, has opinions about edge cases, and can ask a colleague when confused. An agent has none of this. It has exactly the tokens you provide plus whatever its training internalized.

This creates a fundamental tension:

- **Too little instruction:** The agent falls back on training priors, which may not match your project's conventions. It produces generic code that technically works but doesn't fit.
- **Too much instruction:** The agent's instruction-following quality degrades uniformly across ALL instructions, not just the new ones (HumanLayer). Frontier models reliably follow roughly 150-200 instructions before performance drops. Everything counts — system prompts, hooks, plugins, CLAUDE.md, and user messages.

The craft is finding the precise middle: say exactly what Claude doesn't already know and nothing more.

### The Default Assumption

Anthropic's own skill authoring guide states the core principle bluntly:

> **Default assumption: Claude is already very smart.** Only add context Claude doesn't already have. Challenge each piece of information: "Does Claude really need this explanation?" "Can I assume Claude knows this?" "Does this paragraph justify its token cost?"

This is the single most important principle. A 50-token skill that tells Claude to use `pdfplumber` for text extraction outperforms a 150-token version that first explains what PDFs are. The explanation wastes context on knowledge Claude already has.

### The Critical Test

For every line in your skill, ask: **would Claude make a mistake without this?** If the answer is no, delete the line. This isn't about brevity for its own sake — it's about preserving the context budget for instructions that actually change behavior.

---

## Part 2: Prose Style — How to Write Instructions Agents Follow

### Use Imperative Mood

Write commands, not descriptions. Agents respond to directives.

**Good:** "Run the test suite before committing. Use `pytest -v` for verbose output."

**Bad:** "It would be helpful to run tests before committing. The test suite can be invoked with pytest."

The imperative mood creates a clear action the agent can execute. Descriptive prose forces the agent to infer what you want.

### Be Specific and Literal

Agents interpret literally. The Stack Overflow article on coding guidelines makes this vivid: "approach them in the most bad-faith manner possible; if you can find a way to misinterpret it, rewrite it." Write as if for a very skilled but extremely literal engineer who has never seen your codebase.

**Good:** "Use `bun test` to run tests. Never use `npm test` — it is not configured in this project."

**Bad:** "Use the project's test runner."

The GitHub analysis of 2,500 repos found that specificity was the single strongest predictor of effective agents.md files. Generic descriptions like "helpful coding assistant" fail. Precise instructions like "test engineer who writes tests for React components, follows these examples, and never modifies source code" succeed.

### Use Examples Over Explanations

One real code snippet outperforms three paragraphs of description (GitHub agents.md analysis). This aligns with Anthropic's skill authoring guide, which recommends input/output pairs:

```markdown
## Commit message format

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware

**Example 2:**
Input: Fixed bug where dates displayed incorrectly
Output:
fix(reports): correct date formatting in timezone conversion
```

Examples help Claude understand desired style and detail more clearly than descriptions alone. They also serve as implicit tests — if the skill can't produce output matching the examples, something is wrong.

### Use Consistent Terminology

Choose one term and use it everywhere. The Anthropic guide is explicit:

- Always "API endpoint" — not a mix of "URL," "API route," "path"
- Always "extract" — not a mix of "pull," "get," "retrieve"

Inconsistency forces the agent to determine whether different terms mean different things or are synonyms. That's wasted reasoning capacity.

### Write in Third Person for Descriptions

Anthropic's guide warns that skill descriptions injected into system prompts must use third person. "Processes Excel files and generates reports" — not "I can help you process Excel files." Inconsistent point-of-view causes discovery problems because the description sits alongside system prompt text written in a different voice.

### Avoid Idioms and Ambiguity

The Stack Overflow article recommends writing "as if for non-native English speakers." Idioms, sarcasm, and cultural references create unnecessary interpretation work. "Don't reinvent the wheel" is less clear than "Use existing library functions instead of writing custom implementations."

---

## Part 3: Structure — How to Organize a Skill

### The Progressive Disclosure Pattern

This is the single most important structural principle, emphasized by both Anthropic's official docs and Fowler's context engineering article. The idea: SKILL.md is a table of contents that points to detailed materials loaded only when needed.

```
my-skill/
  SKILL.md           # Overview and navigation (< 500 lines)
  reference.md       # Detailed API docs (loaded when needed)
  examples.md        # Usage examples (loaded when needed)
  scripts/
    validate.py      # Utility script (executed, not loaded)
```

This matters because context is a public good. Your skill shares the context window with everything else — system prompt, conversation history, other skills, the user's actual request. Large reference docs loaded eagerly crowd out the conversation.

**Keep references one level deep.** Claude partially reads files referenced from other referenced files (using `head -100` previews). SKILL.md should link directly to every reference file, never through intermediary files.

### Front-Load the Key Use Case

Descriptions longer than 250 characters are truncated. Put the most important keywords and triggers first. For longer reference files (100+ lines), include a table of contents at the top so Claude can see scope even with partial reads.

### The Three-Tier Boundary Pattern

The GitHub analysis found that effective agents.md files structure restrictions using three tiers:

- **Always do** — required behaviors, no questions asked
- **Ask first** — require human approval before proceeding
- **Never do** — absolute prohibitions

This maps directly to Claude Code's permission system (`allowed-tools` for "always," default prompting for "ask first," and deny rules for "never").

### Section Ordering

Based on the patterns observed across sources:

1. **Quick start / essential commands** — what Claude needs for the most common case
2. **Architecture and structure** — the map of the codebase
3. **Conventions and patterns** — how things are done here (with examples)
4. **Boundaries** — what to avoid, what requires approval
5. **Advanced reference** — linked files for deep detail, loaded on demand

Put commands and executable information early. The GitHub analysis found that placing relevant executable commands with specific flags in early sections was a common pattern in effective files.

---

## Part 4: What to Include and What to Omit

### Include: What Makes This Project Different

Anthropic's CLAUDE.md creation prompt captures this perfectly: document "the big picture architecture that requires reading multiple files to understand" — not surface-level details Claude can discover by reading the code.

**Include:**
- Tech stack with versions ("React 18 with TypeScript, Vite, and Tailwind CSS" — not just "React project")
- Build, test, and lint commands with exact flags
- Project structure for monorepos (what lives where and why)
- Non-obvious conventions that differ from defaults
- Domain-specific knowledge Claude couldn't infer from code alone

### Omit: What Claude Already Knows

Anthropic's CLAUDE.md creation prompt explicitly forbids:
- "Provide helpful error messages" (obvious)
- "Write unit tests" (generic)
- "Never include sensitive information" (universal)
- Listing every component or file structure that can be easily discovered
- Generic development practices

The HumanLayer guide adds: don't use CLAUDE.md as a linter replacement. Code style guidelines waste context window. Use deterministic tools (formatters, linters) for things that can be checked mechanically.

### Omit: Time-Sensitive Information

From Anthropic's guide:

**Bad:** "If you're doing this before August 2025, use the old API."

**Good:** Document the current method prominently. Put deprecated patterns in a collapsible "Old patterns" section.

### The Instruction Budget

HumanLayer quantifies the constraint: Claude Code's system prompt already contains ~50 instructions. With a reliable following capacity of 150-200 instructions, you have roughly 100-150 instruction slots for your CLAUDE.md, skills, hooks, and plugins combined. Every instruction you add dilutes every other instruction, including your most important ones.

This is why the advice converges on brevity:
- Anthropic: SKILL.md under 500 lines, descriptions under 250 characters
- HumanLayer: CLAUDE.md under 300 lines, ideally under 60
- Cursor: Individual rule files under 500 lines

---

## Part 5: Degrees of Freedom — When to Be Precise vs. When to Be Loose

Anthropic's skill authoring guide introduces a critical framework: match specificity to the task's fragility.

### High Freedom (text-based guidance)

Use when multiple approaches are valid and context determines the best one. Example: code review instructions. You provide the criteria; Claude adapts to the specific code.

### Medium Freedom (pseudocode or parameterized templates)

Use when a preferred pattern exists but some variation is acceptable. Example: report generation with a template structure but flexible content.

### Low Freedom (exact scripts, no parameters)

Use when operations are fragile and error-prone, or consistency is critical. Example: database migrations that must run in exact sequence. "Run exactly this script. Do not modify the command or add additional flags."

The analogy from Anthropic: **narrow bridge with cliffs** (one safe path — be precise) vs. **open field** (many valid paths — give direction, trust Claude).

---

## Part 6: Testing That Your Skill Works

### Evaluation-Driven Development

Anthropic's skill authoring guide is emphatic: **create evaluations BEFORE writing extensive documentation.** The process:

1. Run Claude on representative tasks without the skill. Document specific failures.
2. Build three test scenarios targeting those gaps.
3. Measure baseline performance without the skill.
4. Write minimal instructions addressing the gaps.
5. Test, compare, refine.

This prevents the most common failure: documenting imagined requirements rather than solving real problems.

### The Two-Claude Pattern

Anthropic recommends working with "Claude A" (the skill author) and "Claude B" (the skill user):

1. Complete a task with Claude A using normal prompting. Notice what context you repeatedly provide.
2. Ask Claude A to capture the reusable pattern as a skill.
3. Test the skill with Claude B (a fresh instance) on related tasks.
4. Observe where Claude B struggles. Return to Claude A with specifics.
5. Iterate.

This works because Claude understands both how to write effective agent instructions and what information agents need. The fresh instance reveals gaps that your familiarity with the project would mask.

### Test Across Models

What works for Opus may need more detail for Haiku. If your skill will be used across model tiers, test with each one. Anthropic's checklist requires testing with Haiku, Sonnet, and Opus.

### The Adversarial Test

From Stack Overflow: approach your guidelines "in the most bad-faith manner possible." If you can find a way to misinterpret an instruction, an agent eventually will. Rewrite until misinterpretation requires deliberate bad faith.

### The Gold Standard File Pattern

Stack Overflow describes creating a comprehensive example file demonstrating all guidelines working together. Individual examples are unit tests; the gold standard is the end-to-end test. It shows what "correct" looks like holistically, not just rule-by-rule.

---

## Part 7: Common Mistakes

### 1. Over-Specification

The most common mistake. Practitioners stuff every possible convention into CLAUDE.md and wonder why Claude ignores instructions. As instruction count rises, compliance drops uniformly across ALL instructions.

**Fix:** Apply the critical test ruthlessly. Delete anything Claude would do correctly without the instruction.

### 2. Explanation Where Direction Is Needed

Skills that explain "why" at the expense of "what" and "how." Background context has a place, but the agent needs actionable instructions first.

**Fix:** Lead with the directive. Add explanation only if it changes behavior. "Use pdfplumber for text extraction" is sufficient. "PDFs are a common document format..." is waste.

### 3. Vague Boundaries

"Be careful with the database" gives no actionable constraint. "Never run DROP or TRUNCATE without explicit user approval" does.

**Fix:** Use the three-tier boundary pattern. Make every prohibition specific enough that compliance is binary — the agent either did or didn't follow it.

### 4. Inconsistent Terminology

Using "endpoint," "route," "path," and "URL" interchangeably. The agent cannot know these are synonyms in your context.

**Fix:** Pick one term. Use it everywhere. Define it once if it's non-obvious.

### 5. Duplicating What Tools Enforce

Writing "always format code with Prettier" in CLAUDE.md when you have a pre-commit hook running Prettier. The hook is deterministic (100% enforcement); the instruction is advisory (~80%). The instruction wastes budget.

**Fix:** Use hooks for things that must happen every time. Use instructions for things that require judgment.

### 6. Copying Other People's Rules

Fowler's context engineering article warns against copying extensive rule sets from others. Context alignment between the rule writer and rule reader matters — what's critical for one team is noise for another.

**Fix:** Build rules iteratively from your own experience. When Claude makes a mistake, add an instruction to prevent it. This produces a minimal, battle-tested ruleset.

### 7. Ignoring the Relevance Filter

Claude Code injects a system reminder: "this context may or may not be relevant to your tasks." The more non-universal information in CLAUDE.md, the more likely Claude downgrades the entire file's relevance. Task-specific instructions that only apply to some sessions actively harm the instructions that apply to all sessions.

**Fix:** Move task-specific instructions into skills (loaded on demand) rather than CLAUDE.md (loaded always).

---

## Part 8: What Anthropic's Own Prompts Teach Us

The Claude Code source leak (March 2026) revealed how Anthropic structures instructions for their own agent. Key patterns from the Piebald-AI reconstruction of 110+ prompt components:

### Modular Composition

Anthropic doesn't use a single monolithic prompt. They assemble from ~110 components conditionally included based on environment, session state, and user interaction. Categories include:

- **Agent prompts** (~30): Instructions for specialized sub-agents (Explore, Plan, Task)
- **Data prompts** (~25): Embedded reference documentation (SDK patterns, API specs)
- **System prompt components** (~60): Core operational instructions
- **System reminders** (~40): Contextual notifications injected during sessions

### Instruction Style

The actual system prompt components use:
- Imperative mood: "Prefer native tools (Write, Edit, Read) over bash"
- Specific prohibitions: "Never modify the git config"
- Behavioral framing through system prompts: "Do not rubber-stamp weak work" (coordinatorMode.ts)
- Conditional filtering: instructions included or excluded based on current mode

### The Lesson

Even Anthropic doesn't try to put everything in one prompt. They modularize aggressively, include conditionally, and keep each component focused on one concern. This is the same progressive disclosure pattern they recommend for skills — because it works on their own system prompts.

---

## Part 9: The Feedback Loop — Making Skills Self-Improving

### Error-Driven Refinement

The Stack Overflow article describes the flywheel: "If it makes a mistake, then they'll go and update that and try to get it to be a flywheel." Agent failures are guideline feedback. Each mistake reveals a tacit convention you hadn't documented.

### Self-Updating Instructions

Several practitioners recommend telling Claude to update CLAUDE.md itself when it encounters undocumented conventions. Over time, the file becomes a living record of the codebase's quirks, written by the same tool that consumes it.

### Observe Navigation Patterns

Anthropic recommends watching how Claude actually uses your skill:
- Does it read files in an unexpected order? Your structure isn't intuitive.
- Does it miss references to important files? Your links aren't prominent enough.
- Does it repeatedly read the same file? That content should be in SKILL.md.
- Does it never access a bundled file? That file is unnecessary.

---

## Part 10: Synthesis — The Principles

These ten principles distill the research into a minimal actionable set:

1. **Only add what Claude doesn't already know.** The critical test for every line: would Claude make a mistake without this?

2. **Be imperative and specific.** Write commands, not descriptions. Include exact flags, exact file paths, exact conventions. If an instruction can be misinterpreted, it will be.

3. **Show, don't tell.** One input/output example outperforms paragraphs of explanation. A gold standard file outperforms dozens of individual rules.

4. **Respect the budget.** ~150-200 instruction capacity total. Every instruction dilutes every other instruction. Be ruthless about what earns its place.

5. **Use progressive disclosure.** SKILL.md is a table of contents. Detailed reference lives in separate files loaded on demand. Keep references one level deep.

6. **Match precision to fragility.** High freedom for judgment calls, low freedom for fragile operations. The bridge/field analogy.

7. **Use deterministic tools for deterministic checks.** Hooks for formatting and linting (100% enforcement). Instructions for architectural judgment (advisory). Don't waste instruction budget on things tools can enforce.

8. **Test with the adversarial mindset.** Approach your instructions in the most bad-faith way possible. If you can misinterpret them, rewrite them.

9. **Build iteratively from real failures.** Don't copy others' rules. Don't anticipate problems. When Claude makes a mistake, add the instruction that prevents it. The result is minimal and battle-tested.

10. **Modularize and scope.** Universal instructions in CLAUDE.md. Task-specific instructions in skills. Project-wide conventions in rules. The right instruction in the right place at the right time.

---

## Appendix: Source Index

| Source | Key Contribution |
|--------|-----------------|
| Anthropic skill authoring best practices (platform.claude.com) | Degrees of freedom framework, progressive disclosure, evaluation-driven development, two-Claude testing pattern |
| Anthropic Claude Code skill docs (code.claude.com) | Skill format spec, frontmatter options, invocation control, subagent execution |
| GitHub blog — 2,500 agents.md analysis (Matt Nigh) | Specificity as #1 predictor, three-tier boundaries, six core areas, examples over explanations |
| Addy Osmani — "Good Spec for AI Agents" | Five principles (vision first, structured specs, modular prompts, self-checks, iterate), curse of instructions, spec-driven development phases |
| Martin Fowler — "Context Engineering for Coding Agents" | Instructions vs. guidance distinction, context as public good, conditional loading, illusion of control caveat |
| Stack Overflow — "Coding Guidelines for AI" | Adversarial testing, gold standard file pattern, error-driven flywheel, write for bad-faith interpretation |
| HumanLayer — "Writing a Good CLAUDE.md" | WHY/WHAT/HOW framework, 150-200 instruction budget, relevance filter, periphery bias, progressive disclosure via linked files |
| Piebald-AI — Claude Code system prompts | 110+ modular components, conditional assembly, imperative style in production prompts |
| Cursor best practices | Composable rule files under 500 lines, gradual implementation, AGENTS.md as cross-tool standard |
| Anthropic — CLAUDE.md creation prompt | Specificity over generality, omit obvious instructions, document architecture not file listings |
