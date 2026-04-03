# Anthropic Skill Analysis: The Craft of Skill Writing

Analysis of all 17 skills in [github.com/anthropics/skills](https://github.com/anthropics/skills), with deep focus on skill-creator, frontend-design, claude-api, pdf, mcp-builder, and docx.

---

## 1. Structural Patterns

### File Layout

Every skill follows the same convention:

```
skill-name/
  SKILL.md          # Required. The only file that's always present.
  LICENSE.txt        # Present on all published skills
  scripts/           # Optional. Python/JS tools the model can execute.
  references/        # Optional. Docs loaded on-demand (progressive disclosure).
  agents/            # Optional. Instructions for subagent delegation (skill-creator only).
  assets/            # Optional. Templates, fonts, HTML viewers.
```

There is no `README.md`. No `config.json`. No manifest beyond the YAML frontmatter in SKILL.md. The template skill is three lines:

```yaml
---
name: template-skill
description: Replace with description of the skill and when Claude should use it.
---
# Insert instructions below
```

### YAML Frontmatter

Every SKILL.md starts with exactly two required fields:

```yaml
---
name: kebab-case-identifier
description: Paragraph-length trigger description.
---
```

No version field. No author field. No dependencies list. The `description` field is the primary triggering mechanism -- the thing that determines whether Claude activates the skill. This is the single most important piece of text in the entire skill.

---

## 2. Skill-by-Skill Analysis

### skill-creator (485 lines, 33KB) -- The Meta-Skill

**Opening lines:** "A skill for creating new skills and iteratively improving them." Then immediately launches into a high-level process overview as a bullet list.

**Voice:** Conversational, almost casual. Uses "Cool? Cool." at one point. Addresses Claude directly as a collaborator: "Your job when using this skill is to figure out where the user is in this process and then jump in." Uses second person throughout.

**Structure:**
- Narrative overview of the full lifecycle (not steps, just a mental model)
- "Communicating with the user" section addressing tone calibration
- Creating a skill (interview, write, test)
- Running and evaluating test cases (5-step continuous sequence)
- Improving the skill (theory of how to think about improvements)
- Description optimization (multi-step with scripts)
- Platform-specific adaptations (Claude.ai, Cowork)
- Reference file pointers

**Key observations:**
- Longest skill by far. It earns the length because it orchestrates a complex multi-phase workflow with subagents, viewers, and iteration loops.
- Explains *why* extensively. The "How to think about improvements" section is 4 numbered paragraphs of reasoning philosophy, not commands. E.g.: "Try hard to explain the **why** behind everything you're asking the model to do. Today's LLMs are *smart*."
- Explicitly warns against heavy-handed MUSTs: "If you find yourself writing ALWAYS or NEVER in all caps, or using super rigid structures, that's a yellow flag."
- Contains a "Skill Writing Guide" subsection -- this is the canonical style guide embedded inside the skill-creator. Key rules from it:
  - Keep SKILL.md under ~500 lines
  - Three-level progressive disclosure: metadata (~100 words) -> SKILL.md body -> bundled resources
  - Reference files clearly from SKILL.md with guidance on *when* to read them
  - Organize by domain variant, not by type
- Platform-aware: different sections for Claude Code, Claude.ai, and Cowork environments.
- Repeats its core loop three times (beginning, middle, end) for emphasis.

### frontend-design (42 lines, 4.4KB) -- Most Popular Skill

**Opening line:** "This skill guides creation of distinctive, production-grade frontend interfaces that avoid generic 'AI slop' aesthetics."

**Voice:** Directive and opinionated. Uses bold assertions: "CRITICAL", "NEVER", "IMPORTANT". This skill is the exception that proves the rule about avoiding heavy-handed MUSTs -- aesthetic direction *needs* strong guardrails because the default mode produces homogeneous output.

**Structure:**
- One-paragraph mission statement
- "Design Thinking" section (4 bullet points: Purpose, Tone, Constraints, Differentiation)
- "Frontend Aesthetics Guidelines" with 5 focus areas
- Anti-pattern list (what NEVER to do)
- Closing philosophy paragraph

**Key observations:**
- Entire skill is a single file. No scripts, no references, no progressive disclosure. 42 lines total.
- Succeeds by being a *mindset shift* rather than a procedure. It doesn't tell Claude how to write CSS -- it tells Claude how to *think about design*.
- The anti-patterns section is as important as the guidelines. It names specific fonts (Inter, Roboto, Arial), specific clichés (purple gradients on white), and says "NEVER converge on common choices (Space Grotesk, for example) across generations."
- Ends with encouragement: "Claude is capable of extraordinary creative work. Don't hold back."

### claude-api (262 lines, 20KB) -- Technical Reference Skill

**Opening line:** "This skill helps you build LLM-powered applications with Claude."

**Voice:** Technical, precise, structured. Uses tables extensively. Imperative instructions with clear scoping: "ALWAYS use claude-opus-4-6 unless the user explicitly names a different model. This is non-negotiable."

**Structure:**
- Defaults section (model, thinking, streaming)
- Language Detection (5-step decision tree with file-extension mapping)
- "Which Surface Should I Use?" (table + decision tree + checklist)
- Architecture overview (one paragraph)
- Current Models table
- Thinking & Effort quick reference
- Compaction quick reference
- Prompt Caching quick reference
- Reading Guide (task-based routing to reference files)
- Common Pitfalls

**Key observations:**
- The SKILL.md is a *router*. It contains just enough information to make good decisions, then points to 20+ reference files organized by language and topic.
- Progressive disclosure is the dominant pattern. The Reading Guide has two sections: "Quick Task Reference" (5 lines each, maps task to files) and "Full File Reference" (numbered list with when-to-read guidance).
- Tables are the primary information format for structured decisions (model selection, language detection, surface selection).
- Anticipates stale knowledge explicitly: "The table above is cached. When the user asks..., query the Models API."
- The "Common Pitfalls" section uses specific, testable rules: "budget_tokens is deprecated on Opus 4.6 and Sonnet 4.6 and must not be used."

### pdf (314 lines, 8KB) -- Document Handling Skill

**Opening line:** "This guide covers essential PDF processing operations using Python libraries and command-line tools."

**Voice:** Tutorial-like. Structured as a reference manual with code examples for every operation.

**Structure:**
- Quick Start (5 lines of code)
- Python Libraries section (pypdf, pdfplumber, reportlab) with code for each operation
- Command-Line Tools section (pdftotext, qpdf, pdftk)
- Common Tasks section (OCR, watermark, images, encryption)
- Quick Reference table (task -> tool -> command)
- Next Steps pointers

**Key observations:**
- Almost entirely code examples. The prose is minimal connective tissue between code blocks.
- Organized by *task* (merge, split, extract, create), not by library. The user thinks "I need to merge PDFs" not "I need pypdf."
- Includes a specific gotcha about Unicode subscripts in ReportLab: "Never use Unicode subscript/superscript characters... The built-in fonts do not include these glyphs, causing them to render as solid black boxes." This is the kind of hard-won knowledge that justifies a skill's existence.
- Delegates to FORMS.md and REFERENCE.md for advanced topics. The main SKILL.md covers 80% of use cases.
- Has 8 Python scripts in `scripts/` for form handling, validation, and conversion.

### mcp-builder (236 lines, 9KB) -- Complex Workflow Skill

**Opening line:** "Create MCP (Model Context Protocol) servers that enable LLMs to interact with external services through well-designed tools."

**Voice:** Process-oriented. Uses phase numbering (Phase 1-4) and sub-steps (1.1, 1.2, etc.).

**Structure:**
- Overview (2 sentences)
- Phase 1: Deep Research and Planning
- Phase 2: Implementation
- Phase 3: Review and Test
- Phase 4: Create Evaluations
- Reference Files section with emoji-labeled links

**Key observations:**
- Organized as a linear workflow, not a reference manual. The phases represent a progression.
- Recommends TypeScript over Python with a specific rationale: "high-quality SDK support and good compatibility in many execution environments... Plus AI models are good at generating TypeScript code."
- Uses WebFetch to pull live SDK documentation rather than bundling it. This keeps the skill current without maintenance.
- The evaluation phase (Phase 4) is surprisingly detailed -- 10 evaluation questions, XML format, specific requirements for question quality (independent, read-only, complex, realistic, verifiable, stable).
- Reference files are organized by concern: best practices, language-specific guides, evaluation guide.

### docx (590 lines, 20KB) -- Largest Document Skill

**Opening line:** "A .docx file is a ZIP archive containing XML files."

**Voice:** Precise and technical. Heavy use of code examples and "CRITICAL" warnings for common mistakes.

**Structure:**
- Quick Reference table (3 rows)
- Reading Content
- Creating New Documents (extensive docx-js API reference)
- Editing Existing Documents (3-step: unpack, edit XML, pack)
- XML Reference (tracked changes, comments, images)
- Dependencies list

**Key observations:**
- The "Critical Rules for docx-js" section at the end of the creation section is a concentrated list of 14 specific rules. Each one represents a discovered failure mode.
- Uses the pattern of showing wrong approach then right approach:
  ```
  // WRONG - never manually insert bullet characters
  new Paragraph({ children: [new TextRun("* Item")] })  // BAD
  // CORRECT - use numbering config
  ```
- Bundles substantial infrastructure: XML schemas, pack/unpack scripts, validators, comment insertion tools. The skill replaces what would normally require a document engineering team.
- The XML editing section is remarkably specific about element ordering in `<w:pPr>` and the difference between `<w:t>` and `<w:delText>`. This granularity exists because XML schema violations silently corrupt documents.

---

## 3. Cross-Cutting Patterns

### How They Open

Every SKILL.md opens with a single sentence or short paragraph stating what the skill does. No preamble, no context-setting, no "Welcome to..." framing. The pattern:

| Skill | Opening |
|-------|---------|
| skill-creator | "A skill for creating new skills and iteratively improving them." |
| frontend-design | "This skill guides creation of distinctive, production-grade frontend interfaces..." |
| claude-api | "This skill helps you build LLM-powered applications with Claude." |
| pdf | "This guide covers essential PDF processing operations..." |
| mcp-builder | "Create MCP servers that enable LLMs to interact with external services..." |
| docx | "A .docx file is a ZIP archive containing XML files." |

The docx opening is notable -- it skips what the skill does and starts with the single most important fact about the domain. This works because the frontmatter description already explains the "what."

### Description Field as Trigger

The `description` in frontmatter is long, specific, and deliberately "pushy" (the skill-creator's own word). Examples:

- **pdf:** "Use this skill whenever the user wants to do anything with PDF files. This includes reading or extracting text/tables from PDFs, combining or merging multiple PDFs into one, splitting PDFs apart..." (Lists every operation.)
- **docx:** "Use this skill whenever the user wants to create, read, edit, or manipulate Word documents (.docx files). Triggers include: any mention of 'Word doc', 'word document', '.docx'..." (Lists trigger phrases.)
- **claude-api:** "TRIGGER when: code imports anthropic/@anthropic-ai/sdk/claude_agent_sdk, or user asks to use Claude API... DO NOT TRIGGER when: code imports openai/other AI SDK..." (Explicit positive and negative triggers.)

The pattern: descriptions are 50-150 words. They enumerate specific use cases. Several include explicit negative triggers (when NOT to use the skill).

### Progressive Disclosure

Three tiers, consistently applied:

1. **Always loaded:** Name + description (~100 words). This is the trigger.
2. **Loaded on activation:** SKILL.md body. The skill-creator recommends <500 lines.
3. **Loaded on demand:** Reference files, read when specific sections are needed.

The claude-api skill is the purest example: SKILL.md is a decision tree that routes to 20+ reference files. The pdf skill is simpler: SKILL.md covers 80% of cases, REFERENCE.md and FORMS.md handle the rest.

Some skills have no tier 3 at all (frontend-design, brand-guidelines, internal-comms). These are short enough that everything fits in SKILL.md.

### Code Examples

Skills that involve code use a consistent pattern:

1. **Quick start** -- minimal working example, usually 3-8 lines
2. **Task-organized sections** -- each common operation gets its own code block
3. **Wrong/Right pairs** -- for common mistakes, show the bad way then the good way
4. **Quick reference table** -- task-to-tool-to-command mapping

Code examples never include unnecessary comments or verbose variable names. The xlsx skill explicitly states: "Write minimal, concise Python code without unnecessary comments."

### What They DON'T Include

Notable omissions across all skills:

- **No installation instructions** beyond one-line package install. Skills assume the environment is ready.
- **No version numbers** in SKILL.md (except the claude-api model table, which is explicitly marked "cached").
- **No error handling boilerplate** in code examples. Examples show the happy path.
- **No "Getting Started" sections.** They jump straight into the work.
- **No changelogs or history.**
- **No badges, shields, or status indicators.**
- **No table of contents** in SKILL.md itself (though the skill-creator recommends TOCs for reference files >300 lines).
- **No explicit audience statement.** They don't say "this skill is for..." -- the description handles targeting.

### Tone and Voice

Two distinct voices emerge across the corpus:

**Conversational-directive** (skill-creator, frontend-design, algorithmic-art, canvas-design): Addresses Claude as a creative collaborator. Uses enthusiasm and encouragement. Explains reasoning. Allows flexibility ("Of course, you should always be flexible...").

**Technical-precise** (claude-api, pdf, docx, xlsx, mcp-builder): Addresses Claude as an executor. Uses tables, decision trees, and code blocks. States rules flatly. Marks exceptions with CRITICAL/IMPORTANT.

The split correlates with the skill's nature: creative/subjective skills use the first voice, procedural/deterministic skills use the second.

### How They Handle Edge Cases

Three strategies, often combined:

1. **Decision trees** -- claude-api's language detection, mcp-builder's transport selection, webapp-testing's approach selection. Rendered as indented text or ASCII diagrams.
2. **Anti-pattern lists** -- frontend-design's "NEVER" list, docx's "Critical Rules," claude-api's "Common Pitfalls." Always framed as what NOT to do, with the correct alternative.
3. **Conditional sections** -- skill-creator has entire sections for "Claude.ai-specific instructions" and "Cowork-Specific Instructions." The doc-coauthoring skill has "If access to sub-agents is available" vs "If no access to sub-agents" branches.

### Use of Scripts

Scripts serve three purposes:

1. **Deterministic operations** that would be error-prone for the model to generate each time (docx pack/unpack/validate, pdf form filling, xlsx formula recalculation).
2. **Infrastructure** the model orchestrates but shouldn't reinvent (skill-creator's eval runner, benchmark aggregator, description optimizer).
3. **Validation** to catch mistakes after generation (docx XML validation, xlsx recalc with error reporting).

The webapp-testing skill captures the philosophy: "Always run scripts with `--help` first. DO NOT read the source until you try running the script first... These scripts can be very large and thus pollute your context window. They exist to be called directly as black-box scripts."

---

## 4. The Implicit Style Guide

Synthesizing across all skills, these are the unwritten rules:

### Structure
1. YAML frontmatter with `name` and `description` only.
2. Open with one sentence stating what the skill does.
3. Organize by task/workflow, not by API/library/component.
4. Use tables for decision matrices and quick references.
5. Use code blocks liberally for anything procedural.
6. End with pointers to reference files or next steps.

### Descriptions
7. Descriptions are 50-150 words and enumerate specific triggers.
8. Include negative triggers for skills that overlap with others.
9. Err on the side of "pushy" -- Claude under-triggers by default.

### Progressive Disclosure
10. SKILL.md covers 80% of use cases in <500 lines.
11. Reference files handle the remaining 20%, loaded on demand.
12. Scripts execute as black boxes -- the model doesn't need to read their source.

### Voice
13. Match voice to skill nature: conversational for creative, precise for procedural.
14. Explain *why* rather than stacking rigid rules. Claude is smart enough to generalize from reasoning.
15. When rules are necessary (format constraints, API gotchas), state them flatly and specifically.
16. Use CRITICAL/IMPORTANT sparingly, only for things that silently break.

### Examples
17. Show wrong/right pairs for common mistakes.
18. Keep code minimal -- no unnecessary comments, no verbose names.
19. Start with a quick-start example, then expand by task.

### What to Omit
20. No installation guides, version numbers, changelogs, TOCs (in SKILL.md), audience statements, or getting-started sections.
21. No meta-commentary about the skill itself ("This skill will help you...") beyond the opening line.

### The Deepest Pattern
22. A skill encodes *judgment*, not just *knowledge*. The pdf skill doesn't just list pypdf functions -- it tells you which tool to reach for given a task. The frontend-design skill doesn't teach CSS -- it shifts Claude's aesthetic posture. The claude-api skill doesn't document the API -- it routes you through decision trees to the right approach. The value of a skill is the decisions it makes for you.

---

## 5. Quantitative Summary

| Skill | SKILL.md Lines | SKILL.md Bytes | Reference Files | Scripts | Total Files |
|-------|---------------|----------------|-----------------|---------|-------------|
| skill-creator | 485 | 33KB | 1 | 7 | 18 |
| frontend-design | 42 | 4.4KB | 0 | 0 | 2 |
| claude-api | 262 | 20KB | 20 | 0 | 27 |
| pdf | 314 | 8KB | 2 | 8 | 12 |
| mcp-builder | 236 | 9KB | 4 | 3 | 10 |
| docx | 590 | 20KB | 0 | 50+ | 55+ |
| xlsx | 291 | 12KB | 0 | 2+ | 5+ |
| algorithmic-art | 404 | -- | 0 | 0 | 3 |
| doc-coauthoring | 375 | -- | 0 | 0 | 2 |
| webapp-testing | 95 | -- | 0 | 1 | 4 |
| brand-guidelines | 73 | -- | 0 | 0 | 2 |
| internal-comms | 32 | -- | 0 | 0 | 2+ |
| canvas-design | 129 | -- | 0 | 0 | 2 |

The range is enormous: 32 lines (internal-comms) to 590 lines (docx). The length correlates with domain complexity, not quality. A 42-line skill (frontend-design) is the most installed of all.
