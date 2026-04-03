# Community Skill Analysis: What Makes Great Skills Great

Deep analysis of the most successful community-built skills for Claude Code and other coding agents. Based on full source reading of five distinct skill collections, each representing a different philosophy and domain.

## Collections Studied

| Collection | Author/Org | Domain | Skills Count | Distinguishing Feature |
|---|---|---|---|---|
| **Superpowers** | Jesse Vincent (obra) | Software methodology | 15+ | Behavioral discipline through pressure-tested prose |
| **Layered Rails** | Vladimir Dementyev (palkan) | Rails architecture | 1 skill + 6 commands + 2 agents | Deep domain expertise encoded as reviewable checklists |
| **HashiCorp Agent Skills** | HashiCorp | Terraform/Packer IaC | 11 | Official vendor skills with reference architecture |
| **BMAD Method** | bmad-code-org | Multi-agent SDLC | 40+ | Persona-as-skill with capability menus |
| **Remotion Best Practices** | Remotion team | Video-in-React | 1 hub + 25 rule files | Thin hub with lazy-loaded domain rules |

---

## 1. Superpowers (obra/superpowers)

**42K+ GitHub stars. Accepted into Anthropic marketplace Jan 2026.**

### Architecture

```
skills/
  brainstorming/SKILL.md          # ~200 lines, process skill
  writing-plans/SKILL.md          # ~150 lines, process skill
  executing-plans/SKILL.md        # ~60 lines, lightweight coordinator
  subagent-driven-development/    # ~250 lines, orchestration skill
  test-driven-development/        # ~300 lines, discipline skill
  systematic-debugging/           # ~350 lines, discipline skill
  verification-before-completion/ # ~120 lines, discipline skill
  dispatching-parallel-agents/    # ~150 lines, technique skill
  writing-skills/                 # ~500 lines, meta-skill
  ...
commands/
  brainstorm.md, execute-plan.md, write-plan.md
agents/
  code-reviewer.md
hooks/
  hooks.json, session-start
```

### Prose Pattern: Imperative + Adversarial

The defining feature of Superpowers is that skills are written *against* the agent. They assume Claude will rationalize skipping steps and preemptively close every loophole. This is unique among all skills studied.

**The "Iron Law" pattern** -- a non-negotiable rule stated as a code block:

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST  
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

**Rationalization tables** -- every excuse the agent might generate, paired with a rebuttal:

| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |

**Red flag lists** -- trigger phrases the agent should recognize as self-deception:

> "I already manually tested it", "Tests after achieve the same purpose", "This is different because..."
> **All of these mean: Delete code. Start over with TDD.**

**The phrase "your human partner"** is deliberate -- it creates a relationship frame where disappointing the human carries weight. The CLAUDE.md explicitly warns contributors not to change this wording.

### Key Structural Patterns

1. **HARD-GATE tags** -- brainstorming skill uses `<HARD-GATE>` XML to mark absolute stopping points
2. **Graphviz flowcharts** -- process skills use `dot` digraphs for decision trees, not decoration
3. **Checklist-as-todo** -- brainstorming says "You MUST create a task for each of these items"
4. **Skill chaining** -- each skill names exactly which skill comes next: "The terminal state is invoking writing-plans"
5. **Announce-at-start** -- skills require the agent to declare which skill it is using

### The Writing-Skills Meta-Skill

The most sophisticated meta-skill in any collection. It treats skill authoring as TDD:

- RED: Run pressure scenario WITHOUT skill, document baseline failures
- GREEN: Write minimal skill addressing those specific rationalizations  
- REFACTOR: Find new loopholes, add counters, re-test

It also defines **Claude Search Optimization (CSO)** -- the insight that the description field determines whether Claude loads the skill at all. Critical finding: *descriptions that summarize workflow cause Claude to follow the summary instead of reading the full skill.*

> "When the description was changed to just 'Use when executing implementation plans with independent tasks' (no workflow summary), Claude correctly read the flowchart and followed the two-stage review process."

### What Makes It Great

- Skills are pressure-tested against real agent behavior, not theoretical
- Adversarial prose that anticipates and blocks rationalization
- Clear skill chaining creates a complete methodology (brainstorm -> plan -> execute -> verify -> finish)
- The writing-skills meta-skill means the system improves itself
- 94% PR rejection rate enforces quality (CLAUDE.md literally warns AI agents not to submit slop)

---

## 2. Layered Rails (palkan/skills)

**By Vladimir Dementyev, author of "Layered Design for Ruby on Rails Applications".**

### Architecture

```
layered-rails/
  .claude-plugin/plugin.json
  agents/
    layered-rails-reviewer.md     # Review agent persona
    layered-rails-gradual.md      # Gradual adoption agent
  commands/
    review.md                     # /layers:review
    analyze.md                    # /layers:analyze  
    analyze-callbacks.md          # /layers:analyze:callbacks
    analyze-gods.md               # /layers:analyze:gods
    spec-test.md                  # /layers:spec-test
    gradual.md                    # /layers:gradual
  skills/layered-rails/
    SKILL.md                      # ~300 lines, hub skill
    references/
      core/architecture-layers.md
      core/extraction-signals.md
      core/specification-test.md
      anti-patterns.md
      patterns/                   # 11 pattern files
      gems/                       # 9 gem reference files
      topics/                     # 8 topic deep-dives
```

### Prose Pattern: Authoritative Reference

Palkan writes like a textbook author encoding his book into machine-readable form. The prose is declarative and categorical -- there are right answers and wrong answers, presented as lookup tables.

**ASCII architecture diagrams:**
```
┌─────────────────────────────────────────┐
│           PRESENTATION LAYER            │
│  Controllers, Views, Channels, Mailers  │
└─────────────────────────────────────────┘
                    ↓
```

**Violation tables** -- pattern matching for code review:

| Violation | Example | Fix |
|-----------|---------|-----|
| Model uses Current | `Current.user` in model | Pass user as explicit parameter |
| Service accepts request | `param :request` in service | Extract value object from request |

**Callback scoring** -- a 1-5 quantitative rubric:

| Type | Score | Keep? |
|------|-------|-------|
| Transformer (compute values) | 5/5 | Yes |
| Operation (business steps) | 1/5 | Extract |

**Grep commands as detection rules:**
```bash
grep -rn "Current\." app/models/
grep -r "request\." app/services/
```

### Key Structural Patterns

1. **Hub-and-spoke** -- SKILL.md is a router with tables of links to reference files
2. **Commands are detailed runbooks** -- `/layers:review` is 200+ lines with exact output format, severity levels, and resolution processes
3. **Agent personas have review methodology** -- the reviewer agent has a 6-step methodology baked in
4. **Allowed-tools in frontmatter** -- SKILL.md restricts itself to `Grep, Glob, Read, Task`
5. **Before/after code pairs** -- every anti-pattern shows the bad code AND the fix
6. **Nuanced classification** -- Current attributes in models are not always violations; the skill distinguishes "acceptable patterns" from "concerning patterns"

### The Review Command

The `/layers:review` command is the standout. It is a complete code review protocol:

1. Identify changed files
2. Determine layers touched
3. Apply layer boundary checks (with grep commands)
4. Run specification test on key files
5. Check for extraction signals
6. Generate review report with severity levels

The output format section defines the exact markdown structure the review should produce, including emoji severity markers and code examples for fixes. This is unusually prescriptive for a skill -- it controls not just what to do but what the output looks like.

### What Makes It Great

- One person's deep domain expertise compressed into machine-executable form
- Quantitative scoring (callbacks 1-5) removes ambiguity from subjective decisions
- Grep commands make detection mechanical, not judgment-based
- The separation of commands, agents, skills, and references is clean
- Nuanced -- knows when Current.user is acceptable vs problematic
- Before/after code for every anti-pattern (agent can copy-paste the fix)

---

## 3. HashiCorp Agent Skills (hashicorp/agent-skills)

**Official vendor skills from HashiCorp for Terraform and Packer.**

### Architecture

```
terraform/
  code-generation/
    .claude-plugin/plugin.json
    skills/
      terraform-style-guide/SKILL.md      # ~250 lines
      terraform-test/SKILL.md             # ~400 lines
      terraform-search-import/SKILL.md    # + references/, scripts/
      azure-verified-modules/SKILL.md
  module-generation/
    skills/
      refactor-module/SKILL.md            # ~350 lines
      terraform-stacks/SKILL.md           # + references/
  provider-development/
    skills/
      new-terraform-provider/SKILL.md     # + assets/
packer/
  builders/skills/                        # 3 builder skills
  hcp/skills/                             # 1 registry skill
```

### Prose Pattern: Vendor Documentation

HashiCorp skills read like official documentation -- precise, comprehensive, neutral. No personality, no adversarial framing, no behavioral psychology. Just the facts.

**Structured with tables throughout:**

| File | Purpose |
|------|---------|
| `terraform.tf` | Terraform and provider version requirements |
| `variables.tf` | Input variable declarations (alphabetical) |

**Good/bad code pairs with brief labels:**
```hcl
# Bad
resource "aws_instance" "webAPI-aws-instance" {}
# Good  
resource "aws_instance" "web_api" {}
```

**Checklists for verification:**
- [ ] Code formatted with `terraform fmt`
- [ ] All variables have type and description
- [ ] No hardcoded credentials or secrets

### Key Structural Patterns

1. **Plugin-per-domain** -- each subdirectory (code-generation, module-generation, provider-development) is a separate Claude plugin with its own `plugin.json`
2. **References directory for heavy content** -- terraform-test has `references/MOCK_PROVIDERS.md`, `CI_CD.md`, `EXAMPLES.md`
3. **Skills reference other skills** -- refactor-module says "Use skill terraform-test" for testing
4. **Input parameters table** -- refactor-module defines formal parameters like an API
5. **Revision history** -- version tracking in the skill itself
6. **External links** -- links to HashiCorp docs as authoritative source

### The Terraform Test Skill

The most complete skill in the collection. It covers:
- Core concepts with definitions
- File structure conventions  
- 12 distinct test patterns (outputs, conditionals, counts, tags, data sources, validation, sequential, parallel, state sharing, providers, complex conditions, cleanup)
- Running commands with flags
- Best practices (11 items)
- Troubleshooting table

Each pattern is a complete, runnable code block. The agent can directly use any pattern without interpretation.

### What Makes It Great

- Official vendor authority -- these are the actual conventions HashiCorp engineers use
- Comprehensive pattern coverage -- 12 test patterns cover nearly every scenario
- Clean separation into installable plugins per workflow
- Scripts directory for executable tools (list_resources.sh)
- Neutral, professional prose that ages well

### What It Lacks

- No behavioral discipline (won't stop an agent from generating bad Terraform)
- No adversarial testing against agent rationalization
- Assumes the agent will follow instructions faithfully
- No process skills -- purely reference material

---

## 4. BMAD Method (bmad-code-org/BMAD-METHOD)

**40+ skills organized as a multi-agent software delivery lifecycle.**

### Architecture

```
src/
  bmm-skills/
    1-analysis/
      bmad-agent-analyst/SKILL.md          # "Mary" persona
      bmad-agent-tech-writer/SKILL.md
      bmad-document-project/SKILL.md       # + templates/, workflows/
      bmad-prfaq/SKILL.md
      bmad-product-brief/SKILL.md
      research/                            # 3 research skills
    2-plan-workflows/
      bmad-agent-pm/SKILL.md               # "James" persona  
      bmad-agent-ux-designer/SKILL.md
      bmad-create-prd/SKILL.md
      ...
    3-solutioning/
      bmad-agent-architect/SKILL.md        # "Winston" persona
      bmad-create-architecture/SKILL.md
      bmad-create-epics-and-stories/SKILL.md
      ...
    4-implementation/
      bmad-agent-dev/SKILL.md              # "Amelia" persona
      bmad-dev-story/SKILL.md
      bmad-code-review/SKILL.md
      bmad-sprint-planning/SKILL.md
      ...
  core-skills/
    bmad-brainstorming/SKILL.md
    bmad-party-mode/SKILL.md               # Multi-agent roundtable
    bmad-help/SKILL.md
    ...
```

### Prose Pattern: Persona + Menu

BMAD's defining innovation is **persona-as-skill**. Each agent skill defines a named character with identity, communication style, and principles:

**Mary (Analyst):** "Speaks with the excitement of a treasure hunter -- thrilled by every clue, energized when patterns emerge."

**Winston (Architect):** "Speaks in calm, pragmatic tones, balancing 'what could be' with 'what should be.'"

**Amelia (Developer):** "Ultra-succinct. Speaks in file paths and AC IDs -- every statement citable. No fluff, all precision."

Each persona presents a **capability menu** on activation:

| Code | Description | Skill |
|------|-------------|-------|
| BP | Expert guided brainstorming facilitation | bmad-brainstorming |
| MR | Market analysis, competitive landscape | bmad-market-research |
| DR | Industry domain deep dive | bmad-domain-research |

### Key Structural Patterns

1. **Phased lifecycle** -- skills organized into analysis -> planning -> solutioning -> implementation
2. **Config-driven** -- all skills load from `_bmad/bmm/config.yaml` for user name, language, output locations
3. **Persona persistence** -- "You must not break character until the user dismisses this persona"
4. **STOP and WAIT gates** -- every persona says "Do NOT execute menu items automatically"
5. **Party Mode** -- spawns actual subagents for each persona in a roundtable discussion, producing genuinely independent perspectives
6. **Manifest-driven** -- agent roster loaded from CSV, not hardcoded

### Party Mode

The most creative skill in any collection. It orchestrates multi-agent discussions where each persona is a real subagent:

> "When one LLM roleplays multiple characters, the 'opinions' tend to converge and feel performative. By spawning each agent as its own subagent process, you get real diversity of thought."

The orchestrator picks 2-4 relevant voices per question, builds context summaries, spawns each as an independent subagent with the agent's full persona, and presents their responses. Agents are instructed to "Disagree with other agents when your expertise tells you to."

### What Makes It Great

- Persona system creates genuine diversity of perspective
- Lifecycle phases give structure to the entire product development process
- Config-driven means the same skills work across projects
- Party Mode is genuinely innovative -- multi-agent deliberation as a skill
- Clean separation between persona skills (who) and capability skills (what)

### What It Lacks

- Verbose -- persona definitions repeat boilerplate across every agent skill
- No adversarial pressure testing visible
- Heavy upfront configuration requirement (`_bmad/bmm/config.yaml`)
- Persona persistence instruction ("do not break character") is fragile

---

## 5. Remotion Best Practices (remotion-dev/skills)

**117K+ weekly installs. Official skill from the Remotion team.**

### Architecture

```
skills/remotion/
  SKILL.md                    # ~60 lines, hub only
  rules/
    animations.md             # ~30 lines
    audio.md
    3d.md
    charts.md
    compositions.md
    fonts.md
    timing.md
    transitions.md
    ... (25 rule files)
    assets/                   # .tsx example files
```

### Prose Pattern: Terse Imperative

Remotion rules are the shortest, most direct prose of any skill studied. The animations rule is under 30 lines:

> "All animations MUST be driven by the `useCurrentFrame()` hook."
> "CSS transitions or animations are FORBIDDEN - they will not render correctly."
> "Tailwind animation class names are FORBIDDEN - they will not render correctly."

One code example. Two absolute prohibitions. Done.

### Key Structural Patterns

1. **Ultra-thin hub** -- SKILL.md is purely a table of contents linking to rule files
2. **Lazy loading** -- "load the [./rules/subtitles.md](./rules/subtitles.md) file for more information"
3. **FORBIDDEN/MUST keywords** -- binary rules with no gray area
4. **One correct way** -- each rule file shows the one right approach, not alternatives
5. **Asset files** -- `.tsx` examples in an `assets/` directory for complex patterns

### What Makes It Great

- Extreme token efficiency -- the hub loads in ~60 tokens, rules load only when needed
- Zero ambiguity -- FORBIDDEN means forbidden, MUST means must
- Domain expertise compressed to essentials -- a Remotion expert's knowledge in 30 lines per topic
- Asset files provide copy-paste-ready code

### What It Lacks

- No process guidance -- purely reference, no methodology
- No decision trees -- assumes you already know what you need
- No anti-patterns or troubleshooting
- Hub skill lacks a description of when NOT to apply rules

---

## Cross-Cutting Analysis

### Pattern 1: Hub-and-Spoke vs Monolithic

Every successful skill collection uses **hub-and-spoke** architecture, but the hub density varies:

| Collection | Hub Density | Spoke Loading |
|---|---|---|
| Remotion | Ultra-thin (60 lines) | Explicit lazy load via links |
| Palkan | Medium (300 lines, tables + principles) | Reference files for depth |
| HashiCorp | Medium (250-400 lines) | References directory |
| Superpowers | Heavy (120-500 lines, full process inline) | Supporting files for tools/prompts |
| BMAD | Medium (persona + menu) | Capability skills as spokes |

**Finding:** Token-efficient hubs with lazy-loaded references outperform monolithic skills. Remotion's 117K installs on a 60-line hub validates this. However, discipline skills (Superpowers) need inline density because the adversarial prose IS the value -- you cannot lazy-load "don't rationalize."

### Pattern 2: Three Skill Archetypes

Every skill falls into one of three categories, each requiring different prose:

**Reference skills** (Remotion, HashiCorp style guide, Palkan pattern catalog):
- Declarative prose: "Use X. Never use Y."
- Tables for lookup
- Code blocks for copy-paste
- Lazy-loadable

**Process skills** (Superpowers brainstorming, writing-plans, BMAD workflows):
- Imperative prose: "You MUST do X before Y"
- Flowcharts for decision points
- Gates and checkpoints
- Skill chaining

**Discipline skills** (Superpowers TDD, verification, debugging):
- Adversarial prose: "If you're thinking X, STOP"
- Rationalization tables
- Red flag lists
- Iron Laws

### Pattern 3: The Description Field Is Critical

Superpowers' writing-skills skill documents the most important discovery:

> Descriptions that summarize workflow create a shortcut Claude will take. The skill body becomes documentation Claude skips.

Good descriptions name triggering conditions, not process:
- "Use when implementing any feature or bugfix, before writing implementation code"
- NOT: "Use for TDD -- write test first, watch it fail, write minimal code, refactor"

This is confirmed by Anthropic's official guidance: "The description is critical for skill selection: Claude uses it to choose the right Skill from potentially 100+ available Skills."

### Pattern 4: Code Blocks Earn Their Place

Every great skill uses code blocks, but differently:

- **Remotion**: Code IS the skill (one correct implementation per topic)
- **Palkan**: Code as violation detector (grep commands) and as before/after pairs
- **HashiCorp**: Code as pattern library (12 test patterns, each runnable)
- **Superpowers**: Code as process illustration (bash commands showing what to run)
- **BMAD**: Code as templates (config structures, prompt templates)

**Finding:** The most useful code blocks are those the agent can use verbatim. Palkan's grep commands and HashiCorp's test patterns are directly executable. Superpowers' bash examples show exact commands to run. Generic pseudocode adds token cost without value.

### Pattern 5: Severity Systems

Skills that do review or analysis define graduated severity:

- **Palkan**: Critical (must fix) / Warning (should fix) / Suggestion (consider) with emoji markers
- **HashiCorp**: Checklist items (binary pass/fail)
- **Superpowers**: Binary (Iron Law violated or not)

**Finding:** Three-level severity with clear criteria works best for review skills. Binary severity works for discipline skills. Checklists work for reference skills.

### Pattern 6: Flowcharts for Decisions, Not Decoration

Superpowers uses Graphviz `dot` digraphs for genuine decision trees:

```dot
digraph when_to_use {
    "Multiple failures?" [shape=diamond];
    "Are they independent?" [shape=diamond];
    ...
}
```

Palkan uses ASCII art for architecture visualization. BMAD uses tables for menus. Remotion uses nothing.

**Finding:** Flowcharts earn their place ONLY at non-obvious decision points. Superpowers' writing-skills skill states this explicitly: "Use flowcharts ONLY for non-obvious decision points. Never use flowcharts for reference material, code examples, or linear instructions."

### Pattern 7: The "When NOT to Use" Signal

The best skills define boundaries:

- Superpowers dispatching-parallel-agents: "Don't use when failures are related"
- Superpowers writing-skills: "Don't create for one-off solutions or standard practices"
- Palkan callback scoring: Some callbacks SHOULD stay (score 4-5)

**Finding:** Skills without boundaries get over-applied. Defining when NOT to use a skill is as important as defining when to use it.

---

## What Separates Great Skills from Adequate Ones

### Great skills:

1. **Assume the agent will resist.** Superpowers' rationalization tables and Iron Laws treat the agent as an adversary to be constrained, not a partner to be guided. This produces dramatically better compliance.

2. **Provide mechanical detection rules.** Palkan's grep commands and callback scoring rubric remove subjectivity. The agent does not need judgment to find violations -- it runs a command and reads the output.

3. **Show the one right way.** Remotion's FORBIDDEN/MUST rules and Palkan's before/after pairs leave no room for interpretation. Great skills are opinionated.

4. **Chain to the next skill.** Superpowers' methodology flows: brainstorm -> write-plan -> execute-plan -> verify -> finish. Each skill names its successor. The agent never wonders "what now?"

5. **Respect the token budget.** Remotion's 60-line hub with lazy-loaded rules is the gold standard for reference skills. Superpowers' discipline skills are long because they must be -- but they earn every line with adversarial prose that actually changes behavior.

6. **Are tested against real agent behavior.** Superpowers' writing-skills meta-skill demands RED-GREEN-REFACTOR for documentation. Palkan's review command was clearly tested against real Rails codebases. HashiCorp's test patterns come from real Terraform workflows.

### Adequate skills:

1. Describe what to do without anticipating resistance
2. Rely on judgment instead of mechanical rules
3. Present multiple options without recommending one
4. Stand alone instead of chaining to a workflow
5. Are verbose without earning the token cost
6. Are written theoretically, not tested empirically

---

## The Implicit Quality Rubric

Extracted from patterns across all five collections:

| Criterion | Weight | Test |
|---|---|---|
| **Triggering precision** | High | Does the description contain ONLY triggering conditions, no workflow summary? |
| **Token efficiency** | High | Could 30% of the content be cut without losing function? If yes, cut it. |
| **Mechanical detectability** | High | Can the agent verify compliance with commands, not judgment? |
| **Boundary definition** | Medium | Does the skill say when NOT to use it? |
| **Adversarial resilience** | Medium | Does it anticipate and block rationalization? (Critical for discipline skills) |
| **Executable examples** | Medium | Can code blocks be used verbatim, or do they require interpretation? |
| **Skill chaining** | Medium | Does it name exactly what comes next? |
| **Hub-and-spoke structure** | Medium | Is heavy reference content lazy-loaded? |
| **Severity graduation** | Low | Does it distinguish critical from advisory findings? |
| **Flowchart restraint** | Low | Are flowcharts used only at genuine decision points? |

### The Single Most Important Insight

**The description field determines whether your skill gets loaded at all.** Every other quality criterion is irrelevant if Claude never reads the skill. Write descriptions that name triggering conditions ("Use when encountering any bug, test failure, or unexpected behavior"). Never summarize the workflow ("Use for systematic debugging with four phases").

### The Second Most Important Insight

**Discipline skills must be written adversarially.** Reference skills can be neutral documentation. Process skills can be friendly checklists. But any skill that tries to constrain agent behavior (TDD, verification, code review standards) must assume the agent will rationalize its way around the constraint and preemptively block every escape route. Superpowers' rationalization tables are not decoration -- they are the core technology.
