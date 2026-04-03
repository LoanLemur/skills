# The Agent Skills Standard

**Purpose:** Document the Agent Skills open standard — specification, adoption, ecosystem, and tool-specific implementations. This fills the gap identified in `00-synthesis.md` which recommends "follow the Agent Skills standard" without documenting what it actually is.

**Sources:** agentskills.io specification, github.com/agentskills/agentskills, Anthropic engineering blog, OpenAI developer blog, Claude Code docs, Codex docs, Simon Willison, VoltAgent registries, Linux Foundation AAIF announcements, and 20+ supporting articles.

---

## 1. Origin and Timeline

**October 16, 2025:** Anthropic launches Agent Skills as a Claude-specific feature. Skills teach Claude repeatable workflows via organized folders of instructions, scripts, and resources. The interactive "skill-creator" skill generates the folder structure and SKILL.md file.

**December 9, 2025:** Anthropic donates Model Context Protocol (MCP) to the Linux Foundation. Anthropic and OpenAI co-found the Agentic AI Foundation (AAIF) alongside Block, with Google, Microsoft, and AWS joining as members. AAIF anchors MCP, Block's goose, and OpenAI's AGENTS.md as founding projects.

**December 18, 2025:** Anthropic opens Agent Skills as a cross-platform standard, publishing the specification at agentskills.io and the reference implementation at github.com/agentskills/agentskills. Launch partners include Atlassian, Canva, Cloudflare, Figma, Notion, Ramp, and Sentry.

**December 19, 2025:** Simon Willison covers the announcement, calling the spec "deliciously tiny" and "quite heavily under-specified." Notes OpenAI's initial absence from the homepage.

**December 20, 2025:** OpenAI adds Agent Skills support to Codex documentation. The Codex logo appears on the agentskills.io homepage.

**February 2026:** AAIF membership crosses 146 organizations (97 new members in a single batch: 18 Gold, 79 Silver). Agent Skills may eventually join AAIF alongside MCP. OpenClaw's ClawHub registry reaches 13,729 community-built skills.

**March 2026:** 16+ major AI tools support the standard. The anthropics/skills repository hits 110k stars.

Sources:
- https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills
- https://siliconangle.com/2025/12/18/anthropic-makes-agent-skills-open-standard/
- https://simonwillison.net/2025/Dec/19/agent-skills/
- https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation

---

## 2. The Specification

The spec lives at https://agentskills.io/specification and https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx. It is intentionally minimal — Willison's "deliciously tiny" is accurate. The entire spec fits in a single page.

### 2.1 Directory Structure

A skill is a directory containing, at minimum, a `SKILL.md` file:

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

### 2.2 SKILL.md Format

YAML frontmatter followed by Markdown content.

**Required fields:**

| Field | Constraints |
|---|---|
| `name` | 1-64 chars. Lowercase alphanumeric + hyphens only. No leading/trailing/consecutive hyphens. Must match parent directory name. |
| `description` | 1-1024 chars. Describes what the skill does AND when to use it. Should include keywords that help agents identify relevant tasks. |

**Optional fields:**

| Field | Constraints |
|---|---|
| `license` | License name or reference to bundled license file. |
| `compatibility` | Max 500 chars. Environment requirements (intended product, system packages, network access). |
| `metadata` | Arbitrary key-value mapping (string keys to string values). |
| `allowed-tools` | Space-delimited list of pre-approved tools. Experimental — support varies by implementation. |

**Minimal example:**
```yaml
---
name: skill-name
description: A description of what this skill does and when to use it.
---
```

**Full example:**
```yaml
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
license: Apache-2.0
metadata:
  author: example-org
  version: "1.0"
---
```

### 2.3 Body Content

No format restrictions. The Markdown body contains skill instructions — step-by-step procedures, examples, edge cases. The spec recommends keeping SKILL.md under 500 lines and moving detailed reference material to separate files.

### 2.4 Optional Directories

**scripts/**: Executable code (Python, Bash, JavaScript). Should be self-contained or clearly document dependencies.

**references/**: Additional documentation loaded on demand. Keep files focused — smaller files mean less context usage.

**assets/**: Static resources (templates, images, schemas, lookup tables).

### 2.5 Progressive Disclosure

The spec defines a three-layer loading model:

1. **Metadata (~100 tokens):** `name` and `description` loaded at startup for all skills
2. **Instructions (< 5,000 tokens recommended):** Full SKILL.md body loaded when skill is activated
3. **Resources (as needed):** Files in scripts/, references/, assets/ loaded only when required

This is the key architectural insight — skills can bundle unlimited context because they don't load everything simultaneously. The design mirrors "a well-organized manual that starts with a table of contents, then specific chapters, and finally a detailed appendix" (Anthropic engineering blog).

### 2.6 File References

Relative paths from the skill root. Keep references one level deep from SKILL.md — avoid deeply nested reference chains.

### 2.7 Validation

The `skills-ref` reference library validates skills:
```bash
skills-ref validate ./my-skill
```

Source: https://agentskills.io/specification

---

## 3. Relationship to MCP

Agent Skills and MCP are complementary, not competing:

- **MCP** = the "plumbing" — defines how an agent connects to databases, APIs, and external tools
- **Agent Skills** = the "manual" / "brain" — teaches the agent how to use those connections effectively

As Anthropic states: Skills "complement Model Context Protocol servers by teaching agents more complex workflows that involve external tools and software." MCP provides tool access; Skills provide procedural knowledge for using those tools.

Both are open standards. MCP is governed by the AAIF (Linux Foundation). Agent Skills is maintained by Anthropic and open to community contributions — it may eventually join AAIF.

---

## 4. Adoption: 16+ Tools

As of March 2026, the following tools support Agent Skills:

| Tool | Vendor | Type |
|---|---|---|
| Claude Code | Anthropic | CLI coding agent |
| Claude (web/mobile) | Anthropic | AI assistant |
| Cursor | Anysphere | AI code editor |
| OpenAI Codex | OpenAI | CLI coding agent |
| Gemini CLI | Google | CLI coding agent |
| Junie | JetBrains | IDE agent |
| GitHub Copilot | GitHub/Microsoft | AI pair programmer |
| VS Code | Microsoft | Code editor |
| OpenHands | Open source | Autonomous agent |
| OpenCode | Open source | CLI coding agent |
| Amp | Sourcegraph | AI coding agent |
| Goose | Block | AI agent |
| Firebender | Firebender | AI coding tool |
| Letta | Letta | Agent framework |
| Mux | Coder | AI agent |
| Autohand | Autohand | AI agent |

The standard replaces tool-specific config files (.cursorrules, CLAUDE.md instructions, etc.) with a portable format. Write once, use everywhere.

Sources:
- https://agentskills.io (homepage logos)
- https://serenitiesai.com/articles/agent-skills-guide-2026

---

## 5. Claude Code Implementation

Claude Code offers the most mature implementation with several extensions beyond the base spec.

### 5.1 Discovery Hierarchy

Skills are loaded from four priority levels (highest wins on name conflict):

| Level | Path | Scope |
|---|---|---|
| Enterprise | Managed settings | All users in organization |
| Personal | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<skill-name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | Where plugin is enabled |

Plugin skills use `plugin-name:skill-name` namespacing to avoid conflicts. Legacy `.claude/commands/` files continue to work but skills take precedence on name collision.

**Monorepo support:** When editing files in subdirectories, Claude Code automatically discovers skills from nested `.claude/skills/` directories (e.g., `packages/frontend/.claude/skills/`).

**Additional directories:** Skills in `--add-dir` directories are loaded automatically with live change detection.

### 5.2 Extended Frontmatter

Claude Code adds these fields beyond the base spec:

| Field | Purpose |
|---|---|
| `disable-model-invocation` | `true` prevents Claude from auto-loading. Manual `/name` only. Default: `false`. |
| `user-invocable` | `false` hides from `/` menu. Background knowledge only. Default: `true`. |
| `argument-hint` | Hint shown during autocomplete (e.g., `[issue-number]`). |
| `context` | `fork` runs skill in an isolated subagent context. |
| `agent` | Subagent type when `context: fork` is set (e.g., `Explore`, `Plan`, `general-purpose`). |
| `model` | Model override when skill is active. |
| `effort` | Effort level override (`low`, `medium`, `high`, `max`). |
| `hooks` | Hooks scoped to skill lifecycle. |
| `paths` | Glob patterns limiting when skill auto-activates. |
| `shell` | Shell for inline commands (`bash` or `powershell`). |

### 5.3 Invocation Model

| Configuration | User can invoke | Claude can invoke | Context loading |
|---|---|---|---|
| Default | Yes | Yes | Description always in context; full skill loads on activation |
| `disable-model-invocation: true` | Yes | No | Description NOT in context; loads only on user invocation |
| `user-invocable: false` | No | Yes | Description always in context; full skill loads on activation |

Descriptions are loaded into context so Claude knows what skills exist, but full content loads only when invoked. Descriptions longer than 250 characters are truncated. The total description budget scales at 1% of context window (fallback: 8,000 characters), configurable via `SLASH_COMMAND_TOOL_CHAR_BUDGET`.

### 5.4 String Substitutions

| Variable | Description |
|---|---|
| `$ARGUMENTS` | All arguments passed when invoking |
| `$ARGUMENTS[N]` / `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing the skill's SKILL.md |

### 5.5 Dynamic Context Injection

The `` !`<command>` `` syntax runs shell commands before skill content reaches Claude. Command output replaces the placeholder — this is preprocessing, not agent execution.

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
---
- PR diff: !`gh pr diff`
- Changed files: !`gh pr diff --name-only`
```

### 5.6 Bundled Skills

Claude Code ships with built-in skills:

| Skill | What it does |
|---|---|
| `/batch <instruction>` | Decomposes work into 5-30 units, spawns one agent per unit in isolated git worktrees, each opens a PR |
| `/claude-api` | Loads Claude API/SDK reference. Auto-activates on `anthropic` imports |
| `/debug [description]` | Enables debug logging, reads session debug log |
| `/loop [interval] <prompt>` | Runs a prompt repeatedly on an interval |
| `/simplify [focus]` | Spawns three parallel review agents, aggregates findings, applies fixes |

### 5.7 Permission Control

Skills can be governed through the permission system:
- `Skill` in deny rules disables all skills
- `Skill(name)` for exact match allow/deny
- `Skill(name *)` for prefix match with any arguments

Source: https://code.claude.com/docs/en/skills

---

## 6. OpenAI Codex Implementation

Codex implements the Agent Skills spec with its own extensions.

### 6.1 Discovery Hierarchy

| Level | Path |
|---|---|
| Repository | `.agents/skills/` in CWD, parent folders, or repo root |
| User | `$HOME/.agents/skills/` |
| Admin | `/etc/codex/skills/` |
| System | Bundled with Codex |

Note the different directory convention: `.agents/skills/` vs Claude Code's `.claude/skills/`. Both read the same SKILL.md format.

### 6.2 agents/openai.yaml

Codex adds an optional metadata file with three sections:

```yaml
interface:
  name: Display Name
  description: What it does
  icon: emoji-or-url
  brand_color: "#hex"
  default_prompt: "suggested starting prompt"

policy:
  allow_implicit_invocation: true  # default

dependencies:
  mcp_servers: [...]
```

The `allow_implicit_invocation` flag (default: `true`) controls whether Codex can auto-activate the skill from user prompts. When `false`, only explicit `$skill` mentions trigger it. This is equivalent to Claude Code's `disable-model-invocation`.

### 6.3 Progressive Disclosure

Codex loads only metadata (name, description, file path, optional openai.yaml data) at startup. Full SKILL.md instructions load only when deciding to use a skill. This matches the spec's three-layer model.

### 6.4 Conflict Handling

When duplicate skill names exist across locations, Codex does NOT merge them — both appear in selectors.

### 6.5 Configuration

Skills can be disabled without deletion:
```toml
# ~/.codex/config.toml
[[skills.config]]
path = "/path/to/skill/SKILL.md"
enabled = false
```

Source: https://developers.openai.com/codex/skills

---

## 7. OpenAI Case Study: Skills for OSS Maintenance

OpenAI published "Using skills to accelerate OSS maintenance" documenting how skills transformed their Agents SDK repositories.

### 7.1 Results

Between December 1, 2025 and February 28, 2026, the two Agents SDK repos merged **457 PRs** (up from 316 in the prior three months — a **44% increase** in merge velocity):
- Python: 182 to 226
- TypeScript: 134 to 231

### 7.2 Architecture

Three components work together:
1. **AGENTS.md** — repository-level mandatory workflow rules (if/then triggers)
2. **Skills in `.agents/skills/`** — repeatable operational patterns
3. **GitHub Actions** — enforce the same workflows in CI

### 7.3 Skills Implemented

**Python repository:**
- `code-change-verification`: formatting, linting, type-checking, tests
- `docs-sync`: audits documentation against codebase
- `examples-auto-run`: executes examples in automated mode
- `final-release-review`: compares release candidates with previous versions
- `openai-knowledge`: pulls current API docs via MCP

**TypeScript repository:**
- `changeset-validation`: validates release metadata and version bumps
- `integration-tests`: tests on multiple runtimes (Node.js, Bun, Deno, Workers)
- `pnpm-upgrade`: coordinates toolchain updates

### 7.4 Design Principles

- **Separation of concerns:** "Interpretation, comparison, and reporting stay with the model" — deterministic shell work goes into scripts
- **Progressive disclosure:** Metadata loads first, full docs only when selected, scripts/references when needed
- **Structured outputs:** Skills like `pr-draft-summary` produce rigid formats (branch names, PR titles, draft descriptions)
- **Mandatory workflows:** AGENTS.md establishes if/then rules requiring specific skills:
  - Before editing runtime changes: use `$implementation-strategy`
  - When code/tests/examples change: run `$code-change-verification`
  - For API work: use `$openai-knowledge`

### 7.5 Key Insight

Integration tests validate examples by comparing actual stdout/stderr against intended behavior encoded in source comments — not relying solely on exit codes. Human review now focuses on architecture and API choices while automated review handles correctness.

Source: https://developers.openai.com/blog/skills-agents-sdk

---

## 8. Ecosystem and Registries

### 8.1 anthropics/skills (Official Repository)

Anthropic's public repository (110k stars, 12.4k forks) contains production skills and reference implementations.

**Categories:**
- Creative & Design (art, music, design)
- Development & Technical (testing, MCP server generation)
- Enterprise & Communication (branding, communications)
- Document Skills (DOCX, PDF, PPTX, XLSX — source-available, not open source)

**Installation methods:**
- Claude Code: `/plugin marketplace add anthropics/skills`
- Claude.ai: upload via UI (paid plans)
- Claude API: Skills API endpoint

Source: https://github.com/anthropics/skills

### 8.2 VoltAgent/awesome-agent-skills

Community-maintained collection of **1,060+ skills** from official dev teams. Curated for quality — "real-world skills created and used by actual engineering teams, not mass AI-generated content."

**Major vendor contributions:**
- **Anthropic:** 17 skills (documents, presentations, design, frontend, MCP)
- **Vercel:** 7 skills (React/Next.js best practices, components, upgrades)
- **Cloudflare:** 6 skills (AI agents, MCP servers, Durable Objects, Workers)
- **Netlify:** 12 skills (Functions, Edge Functions, database, image CDN, forms)
- **HashiCorp:** 11 skills (Terraform providers, modules, testing patterns)
- **Google:** Multiple tracks (Gemini API, Workspace CLI, Labs/Stitch design-to-code)
- **Stripe, Sanity, Firecrawl, Tinybird, Better Auth, Supabase:** Specialized integrations

Skills link to vendor documentation via officialskills.sh for verification.

Source: https://github.com/VoltAgent/awesome-agent-skills

### 8.3 OpenClaw / ClawHub

OpenClaw is a locally-running AI assistant. ClawHub is its public skills registry hosting **13,729 community-built skills** (as of February 28, 2026).

VoltAgent curated **5,211 skills** from ClawHub after filtering out 7,215 due to spam, duplicates, low quality, crypto/finance focus, and security concerns. Categories include:

| Category | Count |
|---|---|
| Coding Agents & IDEs | 1,200+ |
| Web & Frontend Development | 924 |
| DevOps & Cloud | 393 |
| Search & Research | 345 |
| Browser & Automation | 322 |
| Image & Video Generation | 170 |
| Git & GitHub | 167 |
| 12+ other categories | varies |

Note: OpenClaw skills are for its own ecosystem and do not explicitly follow the Agent Skills standard. The overlap is in concept (packaged procedural knowledge for agents) rather than format compliance.

Sources:
- https://github.com/VoltAgent/awesome-openclaw-skills
- https://github.com/openclaw/clawhub

### 8.4 Other Ecosystem Tools

- **agent-skills-cli** (Karanjot786): Universal CLI accessing 40,000+ skills from SkillsMP, syncs to Cursor, Claude Code, Copilot, Codex, and Antigravity
- **agent-skill-creator** (FrancyJGLisboa): Turns any workflow into reusable skills installable on 14+ tools
- **awesome-claude-skills** (travisvn): Curated list focused on Claude Code
- **skills-ref**: Official validation library from agentskills/agentskills repo

---

## 9. Governance

Agent Skills is an **open format maintained by Anthropic** and open to community contributions. The specification repository is at github.com/agentskills/agentskills.

- **Code license:** Apache 2.0
- **Documentation license:** CC-BY-4.0
- **Community:** Discord server (discord.gg/MKPE9g8aUy)

The spec may eventually join the **Agentic AI Foundation (AAIF)** under the Linux Foundation — the same body that governs MCP. AAIF was co-founded by Anthropic, OpenAI, and Block in December 2025. By February 2026 it had 146+ member organizations. Current AAIF projects include MCP, goose (Block), and AGENTS.md (OpenAI).

---

## 10. Design Philosophy and Analysis

### 10.1 Why Minimal

Simon Willison's observation that the spec is "quite heavily under-specified" is both accurate and intentional. The spec defines:
- A directory structure
- A frontmatter schema with 5-6 fields
- A progressive disclosure model
- Nothing else

This minimalism is the spec's strength. It's trivially implementable — any tool that reads YAML frontmatter and Markdown can support Agent Skills. The barrier to adoption is nearly zero, which explains the rapid 16+ tool adoption in three months.

### 10.2 What the Spec Leaves to Implementations

The spec deliberately does not define:
- **Discovery hierarchy** — each tool defines its own paths (`.claude/skills/` vs `.agents/skills/` vs wherever)
- **Invocation semantics** — how skills are triggered (slash commands, auto-detection, explicit references)
- **Tool execution** — what `allowed-tools` means in practice
- **Conflict resolution** — how duplicate names are handled
- **Permission model** — how skill access is governed

Each implementation fills these gaps differently, which is appropriate for a cross-platform standard.

### 10.3 MCP Analogy

The pattern mirrors MCP's success: define the minimum viable protocol, let implementations innovate on top, achieve adoption through simplicity. MCP has 10,000+ published servers. Agent Skills already has 13,000+ in ClawHub alone (though quality varies enormously — the 52% curation rejection rate tells the story).

### 10.4 Progressive Disclosure Is the Key Innovation

The three-layer loading model (metadata -> instructions -> resources) solves the fundamental tension between rich context and limited context windows. A project can have dozens of skills installed without any context cost until they're activated. This is why the spec recommends keeping SKILL.md under 500 lines and moving detail to reference files.

### 10.5 Implications for Skill Authors

From the spec and ecosystem patterns:

1. **The description field is everything for discovery.** Front-load keywords. Be specific about WHEN to use the skill, not just what it does.
2. **Keep SKILL.md under 500 lines.** Move detail to references/.
3. **Scripts should be deterministic.** Let scripts handle mechanical work; let the model handle interpretation.
4. **Name must match directory name.** This is validated by skills-ref.
5. **Test portability.** A skill that only works in Claude Code (using `context: fork`, `agent:`, etc.) is not truly portable. Keep the base skill standard-compliant and use tool-specific extensions as enhancements.

---

## 11. Cross-Reference to Synthesis

The synthesis document (`00-synthesis.md`) references Agent Skills in several places:

- **Part 3, Skill Design Principle 2:** "Skills follow the Agent Skills open standard — portable across Claude Code, Codex, Cursor, Windsurf." This is confirmed — the actual list is 16+ tools, not 4.
- **Part 3, Skill Design Principle 1:** "Skills are lazy-loaded context — only injected when relevant." This maps directly to the spec's progressive disclosure model.
- **Part 1, P4:** "Deterministic gates beat probabilistic instructions." The spec's `allowed-tools` field and Claude Code's `hooks` field in skill frontmatter enable deterministic enforcement within skills.
- **Part 1, P7:** "Context is the fundamental resource." The three-layer loading model is a direct response to this constraint.

The synthesis recommendation to "follow the Agent Skills standard" is well-founded. The standard provides the right level of structure for portable, context-efficient skill packaging.
