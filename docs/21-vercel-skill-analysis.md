# Vercel Skill Analysis: Craft and Quality Patterns

Study of vercel-labs/agent-skills (24K stars) and vercel-labs/skills (the CLI/package manager). Focus on what makes these skills well-written enough to achieve 200K+ installs.

## Sources Studied

- `react-best-practices` (263K installs) — 69 rules across 8 categories
- `web-design-guidelines` (213K installs) — thin dispatcher to live URL
- `composition-patterns` — 8 rules across 4 categories
- `react-native-skills` — 30+ rules across 8 categories
- `react-view-transitions` — reference-based architecture
- `deploy-to-vercel` — operational/procedural skill
- `vercel-cli-with-tokens` — operational/procedural skill
- `find-skills` — meta-skill for discovery
- Vercel plugin for Claude Code — 47+ skills injected via system-reminder
- Skills CLI (`npx skills`) — package manager internals

---

## 1. Two Distinct Skill Archetypes

Vercel's skills split cleanly into two types with radically different structures:

### Type A: Rule Libraries (react-best-practices, composition-patterns, react-native)

Structure: SKILL.md (index) + rules/ directory + AGENTS.md (compiled)

- SKILL.md is a **table of contents**, not a monolith. It categorizes, prioritizes, and provides quick reference — the agent reads this first and drills into individual rule files as needed.
- Individual rules in `rules/` are self-contained: frontmatter (title, impact, tags) + explanation + incorrect/correct code pair.
- AGENTS.md is a **compiled** version concatenating all rules into one file for agents that can only read a single file.

### Type B: Procedural Skills (deploy-to-vercel, vercel-cli-with-tokens)

Structure: SKILL.md (monolithic) + optional resources/ directory with scripts

- SKILL.md contains the entire decision tree inline.
- Heavy use of conditional branching: "If X, do Y. If not X, do Z."
- Explicit environment awareness: different paths for sandbox vs. CLI vs. CI.

### Type C: Dispatcher Skills (web-design-guidelines)

Structure: SKILL.md that points to a live URL

- Extremely thin — the skill's job is to `WebFetch` a URL and apply it.
- The actual rules live externally and update without reinstalling the skill.
- A clever pattern: the skill is a stable pointer to unstable (frequently updated) content.

**Craft insight:** The skill format itself is chosen to match the nature of the knowledge. Rules that are scanned during code review get decomposed. Procedures that are followed step-by-step stay monolithic. Frequently changing content gets a URL pointer.

---

## 2. The Frontmatter Contract

Every SKILL.md uses YAML frontmatter with exactly these fields:

```yaml
---
name: vercel-react-best-practices
description: React and Next.js performance optimization guidelines...
license: MIT
metadata:
  author: vercel
  version: "1.0.0"
---
```

Every rule file uses:

```yaml
---
title: Promise.all() for Independent Operations
impact: CRITICAL
impactDescription: 2-10x improvement
tags: async, parallelization, promises, waterfalls
---
```

**What's notable:**

- `description` in SKILL.md does double duty: it tells the agent when to activate AND tells the human what it does. The description field is written as trigger instructions: "Use when writing, reviewing, or refactoring React/Next.js code..."
- `impact` uses a fixed vocabulary: CRITICAL, HIGH, MEDIUM, LOW. No ambiguity.
- `impactDescription` is quantified when possible: "2-10x improvement", "200-800ms import cost". Not "improves performance" but a number.
- Tags are flat comma-separated strings, not arrays. Lightweight.

**Craft insight:** The frontmatter is designed for machine parsing. The description is an activation condition, not a marketing blurb. Impact levels create a priority queue the agent can sort by.

---

## 3. Prose Style: Imperative, Specific, Opinionated

### Rules never hedge

Not: "You might want to consider using Promise.all() when operations are independent."

Instead: "When async operations have no interdependencies, execute them concurrently using `Promise.all()`."

The voice is imperative and declarative. There is no "consider", "you may want to", or "it's generally a good idea to". The skill tells the agent what to do, not what to think about.

### The Incorrect/Correct pattern is universal

Every rule follows the same structure:

1. One-sentence explanation of why
2. **Incorrect (description of what's wrong):** — code block
3. **Correct (description of what's right):** — code block
4. Optional: additional context, references

The labels are always "Incorrect" and "Correct", never "Bad" and "Good" or "Before" and "After". This is precise — incorrect means the code has a specific defect, not that it's aesthetically unpleasant.

### Explanations in parenthetical annotations

The code block labels carry information:

- `Incorrect (sequential execution, 3 round trips):`
- `Correct (parallel execution, 1 round trip):`
- `Incorrect (imports entire library):`
- `Correct - Next.js 13.5+ (recommended):`

The parenthetical tells the agent WHY the code is incorrect/correct without requiring it to read the full explanation. An agent scanning rules can make decisions from the labels alone.

### Quantified impact everywhere

- "15-70% faster dev boot, 28% faster builds, 40% faster cold starts"
- "up to 10,000 re-exports in their entry file"
- "it takes 200-800ms just to import them"
- "2-10x improvement"

Numbers replace adjectives. Not "much slower" but "200-800ms". Not "significantly improves" but "2-10x".

---

## 4. Structural Patterns

### Priority tables as routing logic

Every rule-library skill opens with a priority table:

```
| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Eliminating Waterfalls | CRITICAL | `async-` |
| 2 | Bundle Size Optimization | CRITICAL | `bundle-` |
```

This serves three functions:
1. Tells the agent what to fix first (priority ordering)
2. Maps file prefixes to categories (the agent can find `async-*.md` files)
3. Communicates impact level without reading each rule

### Quick Reference sections as scannable indexes

Below the priority table, each category gets a bullet list of rule names with one-line descriptions. The agent can scan this to find relevant rules without reading every file. The format is consistent:

```
- `rule-name` - One-line description of what it does
```

### The _template.md pattern

Each rules/ directory contains a `_template.md` showing the exact structure new rules must follow. This is a contributor guardrail — it ensures every rule has the same frontmatter fields, the same section order, the same Incorrect/Correct structure.

### The _sections.md pattern

The composition-patterns skill has `_sections.md` defining all categories, their ordering, impact levels, and descriptions. This is metadata about the rule set itself — a schema for the rules directory. The build tooling reads this to generate AGENTS.md.

### AGENTS.md as compiled output

The react-best-practices skill has a build pipeline (`packages/react-best-practices-build/`) that:
1. Reads `_sections.md` for category definitions
2. Reads all `rules/*.md` files
3. Compiles them into a single `AGENTS.md` (3,750 lines)
4. Validates rule structure against the template
5. Runs test cases

AGENTS.md is a build artifact, not a hand-written file. This is a publishing pipeline for skills.

---

## 5. The "When to Apply" Section

Every SKILL.md has a "When to Apply" (or "When to Use") section written as a bulleted list of situations:

```markdown
## When to Apply

Reference these guidelines when:
- Writing new React components or Next.js pages
- Implementing data fetching (client or server-side)
- Reviewing code for performance issues
```

This is an activation contract. The agent reads this to decide whether to load the skill. The items are specific activities, not vague domains.

**Craft insight:** The activation conditions are task-verbs ("writing", "implementing", "reviewing"), not nouns ("React", "performance"). This helps the agent match on what the human is doing, not what technology they're using.

---

## 6. Framework Version Handling

### Conditional sections with warnings

```markdown
### 4. React 19 APIs (MEDIUM)

> **Warning: React 19+ only.** Skip this section if using React 18 or earlier.
```

The version gate is a blockquote callout placed before the section, not buried in the text. The instruction is binary: skip or apply. No "if you're on React 19, consider...".

### Version-specific code alternatives

The barrel-imports rule shows both approaches:

```
**Correct - Next.js 13.5+ (recommended):**
...optimizePackageImports config...

**Correct - Direct imports (non-Next.js projects):**
...manual deep imports...
```

Multiple "Correct" blocks handle version/framework differences. Each is labeled with its applicability.

### External reference links

Rules link to official documentation and blog posts:
- `Reference: [How we optimized package imports in Next.js](https://vercel.com/blog/...)`
- `Reference: [https://nextjs.org/docs/app/guides/authentication](https://nextjs.org/docs/app/guides/authentication)`

These are at the end of the rule, never inline. They're escape hatches for the agent to fetch more context.

---

## 7. The Procedural Skill Pattern (deploy-to-vercel)

The deploy skill is the most sophisticated procedural skill. Its craft reveals patterns absent from rule libraries:

### Decision tree as numbered steps

```
## Step 1: Gather Project State
## Step 2: Choose a Deploy Method
### Linked + has git remote -> Git Push
### Linked + no git remote -> `vercel deploy`
### Not linked + CLI authenticated -> Link first
### Not linked + CLI not authenticated -> Install, auth, link
### No-Auth Fallback -- claude.ai sandbox
### No-Auth Fallback -- Codex sandbox
```

The headings ARE the decision tree. An agent can route to the correct section by evaluating the conditions in the heading text.

### Explicit "do not" instructions

```
**Do NOT** use `vercel project inspect`, `vercel ls`, or `vercel link`
to detect state in an unlinked directory -- without a `.vercel/` config,
they will interactively prompt (or with `--yes`, silently link as a side-effect).
```

```
**Do not** curl or fetch the deployed URL to verify it works.
Just return the link.
```

These are anti-patterns the agent is likely to attempt. The skill preempts them with explicit prohibitions. The language is unambiguous: "Do NOT" and "Do not", not "avoid" or "prefer not to".

### Environment-specific sections

The deploy skill has separate sections for:
- Claude Code / terminal-based agents
- Sandboxed environments (claude.ai)
- Codex

Each environment gets its own path through the same procedure. The skill doesn't abstract over environments — it branches explicitly.

### Working Agreement sections

The `vercel-cli-with-tokens` skill ends with a "Working Agreement" — a bulleted list of behavioral constraints:

```
- Never pass VERCEL_TOKEN as a --token flag
- Check the environment for tokens before asking the user
- Default to preview deployments
- Ask before pushing to git
```

This is a contract section: not about what to do, but about how to behave while doing it.

---

## 8. The Dispatcher Pattern (web-design-guidelines)

The entire skill is 25 lines. It:
1. Names a URL to fetch
2. Describes what to do with the fetched content
3. Specifies the output format

This is a pointer, not a payload. The actual rules live at `github.com/vercel-labs/web-interface-guidelines`. When rules change, the skill doesn't need updating.

**Craft insight:** This is the thinnest viable skill — proof that a skill doesn't need to contain knowledge, just direct the agent to knowledge.

---

## 9. The Skills CLI: Distribution Mechanics

### Skill discovery

The CLI searches for SKILL.md files in known directories: `skills/`, `.agents/skills/`, `.claude/skills/`, `.cursor/skills/`, etc. It checks 25+ locations covering every major coding agent.

Skills are discovered by walking directories and parsing frontmatter. There's no registry index — the source repo IS the registry. `npx skills add vercel-labs/agent-skills` clones the repo and finds all SKILL.md files within it.

### Installation model

1. Clone repo to temp directory
2. Discover all SKILL.md files within it
3. Copy each skill to a canonical location: `.agents/skills/<skill-name>/`
4. Create symlinks from agent-specific directories (`.claude/skills/`, `.cursor/skills/`) to the canonical location
5. Record metadata in a lock file (`.agents/.skill-lock.json`)

The symlink model means one copy of the files serves all agents. The canonical directory (`.agents/skills/`) is the single source of truth.

### Update detection

Uses GitHub Trees API to compute a folder hash for each skill. Compares against the lock file hash to detect changes. No version bumping required — any change to any file in the skill directory triggers an update.

### Frontmatter requirements

Only two fields are required: `name` and `description`. Everything else is optional. A minimal valid skill is:

```yaml
---
name: my-skill
description: What this skill does
---
# Instructions here
```

This low barrier explains the ecosystem growth — anyone can create a skill in one file.

---

## 10. What Makes 200K+ Install Skills

Synthesizing across all skills studied:

### They solve universal problems

react-best-practices covers patterns every React developer encounters. web-design-guidelines covers rules every web project needs. These aren't niche — they're foundational.

### They're scannable, not readable

No skill is designed to be read top-to-bottom. They're designed to be scanned:
- Priority tables for routing
- Quick reference sections for finding
- Individual rule files for applying
- Compiled AGENTS.md for agents that need everything at once

### They're opinionated with escape hatches

Rules say "do this, not that" with Incorrect/Correct pairs. But they also link to references for the agent (or human) who wants to understand why. The opinion comes first; the justification is available but not required.

### They quantify impact

Every rule has a measurable claim: milliseconds saved, percentage improvement, complexity reduction. This lets agents prioritize automatically and lets humans verify the claims.

### They preempt failure modes

The best skills don't just say what to do — they explicitly list what NOT to do. The deploy skill lists specific CLI commands that will silently break things. The barrel-imports rule explains why tree-shaking doesn't help. Preemptive anti-patterns are a hallmark of mature skills.

### They separate activation from instruction

The SKILL.md frontmatter description and "When to Apply" section are written for the routing layer (deciding whether to load the skill). The actual instructions are written for the execution layer (doing the work). These are different audiences with different needs, and the skills serve both.

### They use consistent internal structure

Every rule follows the same template. Every priority table uses the same columns. Every procedural skill uses the same step numbering. This consistency means an agent that understands one Vercel skill understands all of them — the format is learned once.

---

## 11. Patterns Not Seen Elsewhere

### Build pipelines for skills

The `react-best-practices-build` package compiles individual rules into AGENTS.md with validation. Skills aren't just authored — they're built and tested. The build has test cases that verify rule structure.

### The dual-file pattern (SKILL.md + AGENTS.md)

SKILL.md is the routing/index file. AGENTS.md is the complete payload. This serves different agent architectures: agents that can navigate directories read SKILL.md and drill into rules/. Agents limited to one file read AGENTS.md.

### Impact frontmatter as machine-readable priority

The `impact: CRITICAL` field in rule frontmatter isn't prose decoration. It's a sortable field that an agent can use to prioritize which rules to apply first when reviewing code. Combined with `impactDescription`, it's a machine-readable cost/benefit analysis.

### Prefix-based file naming as category membership

Rule files are named `async-parallel.md`, `bundle-barrel-imports.md`. The prefix IS the category. An agent can `glob("rules/async-*.md")` to find all waterfall rules. No index file needed — the filesystem is the index.

### Plugin manifest for skill grouping

The CLI supports `plugin-manifest.ts` — a way for a parent package to declare which subdirectories contain skills. This means npm packages can bundle skills as a side effect of installation (`npx skills experimental_sync` crawls `node_modules`).

---

## 12. The Implicit Quality Bar

Based on what Vercel publishes vs. what they don't:

1. **Every rule must have an Incorrect/Correct code pair.** No rule is just prose. If you can't show the wrong way and the right way, it's not a rule.

2. **Impact must be quantified or categorized.** "Better performance" is not acceptable. "2-10x improvement" or "CRITICAL" is required.

3. **Instructions must be deterministic.** The agent should produce the same output given the same input. No "use your judgment" — instead, explicit decision trees.

4. **Version constraints must be explicit.** Not "works with newer versions" but "React 19+ only" or "Next.js 13.5+".

5. **Anti-patterns must be documented.** Every "do this" needs a corresponding "don't do this" when the wrong approach is plausible.

6. **External references link to primary sources.** Not blog posts or tutorials, but official docs and the Vercel engineering blog (which carries the same authority as docs for Vercel-specific patterns).

7. **Activation conditions use task verbs.** "Writing new React components" not "React development". The trigger is an activity, not a technology.

8. **The skill must work without human intervention.** No "ask the user to decide" unless there's genuinely no way to determine the answer programmatically. The deploy skill asks about teams only because the choice is inherently subjective.
