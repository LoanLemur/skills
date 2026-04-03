# Claude Code Source Leak: Architectural Insights for Skill Design

**Context:** On March 31, 2026, a 59.8 MB JavaScript source map was accidentally included in `@anthropic-ai/claude-code` v2.1.88 on npm. A missing `.npmignore` entry shipped ~512,000 lines of unobfuscated TypeScript across ~1,900 files. The codebase was mirrored on GitHub within hours (84K+ stars). Boris Cherny (Anthropic engineer) confirmed it was "plain developer error, not a tooling bug."

**Sources:** Engineer's Codex, Alex Kim, WaveSpeedAI, Latent Space, Layer5, Superframeworks, Sabrina Dev, Haseeb Qureshi, DEV Community, Adversa AI, Straiker, SecurityWeek, The Register, Piebald-AI/claude-code-system-prompts. Full URLs in footnotes.

---

## 1. Sub-Agent Execution Models

The leak reveals three distinct execution models for spawning subsidiary agents. Each trades off context sharing, isolation, and cost differently.

### Fork

- Creates a **byte-identical copy** of parent context
- Hits the API's prompt cache — spawning 5 agents costs barely more than 1
- Sub-agents return only output, not full context, preventing state contamination
- Best for: parallel subtasks that need the same context (e.g., reviewing multiple files, running concurrent checks)
- Key insight: **"Parallelism is essentially free. Running a single agent is the unoptimized path."**

### Teammate

- Runs in a **separate tmux/iTerm pane** with its own terminal session
- Communicates via **file-based mailbox** system — agents read/write to shared files
- Each teammate gets its own context window, keeping utilization at ~40% (vs. 80-90% for a single agent)
- Teammates can message each other directly; sub-agents cannot (flat roster, no nesting of teammates)
- A "team lead" session coordinates work, assigns tasks, and synthesizes results
- Best for: independent workstreams that need loose coordination (e.g., frontend + backend of same feature)

### Worktree

- Each agent gets its own **isolated git worktree** with a dedicated branch
- Branches from `origin/HEAD`; unused worktrees cleaned up automatically
- Changes preserved with prompt to keep if work exists
- Best for: exploratory or risky work that might need to be thrown away

### Async Execution Pattern

When `run_in_background: true`, coordinator mode is active, or KAIROS is on, the agent runs asynchronously. `registerAsyncAgent()` fires `runAsyncAgentLifecycle()` as a detached promise. The parent immediately gets an `async_launched` status with the agent's ID and output file path. The async lifecycle handles execution, progress tracking, result finalization, worktree cleanup, and notification enqueuing.

### Skill Design Implication

Skills that dispatch parallel work should prefer the Fork model for read-heavy tasks (reviews, analysis) and the Worktree model for write-heavy tasks (implementation, refactoring). The Teammate model suits multi-session orchestration where humans supervise. Skills should never assume sub-agents share mutable state — they communicate through results, not shared context.

---

## 2. Prompt Caching Architecture

### The SYSTEM_PROMPT_DYNAMIC_BOUNDARY

The system prompt is split by a marker called `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`:

- **Above the boundary (static):** Behavioral instructions, coding style rules, safety guidelines, tool definitions. Cached globally across organizations using Blake2b hashing (~3,000 tokens cached).
- **Below the boundary (dynamic):** User's CLAUDE.md files, MCP server instructions, environment info, git status, current date. Session-specific, never cached.

Anything tagged `DANGEROUS_uncachedSystemPromptSection()` is explicitly marked as cache-breaking, so engineers know the cost before they change it.

### Cache Invalidation

`promptCacheBreakDetection.ts` tracks **14 different cache-invalidation vectors**. "Sticky latches" prevent mode toggles from disrupting the cache. Adding an MCP tool, putting a timestamp in the system prompt, or switching models mid-session can invalidate the entire cache and 5x costs for that turn.

### Economic Reality

Without prompt caching, a long Opus coding session (100 turns with compaction cycles) costs $50-100 in input tokens. With it, $10-19. This is why Claude Code Pro at $20/month is economically viable.

### Skill Design Implication

**Skill text stability is an economic concern, not just a stylistic one.** Skills injected into the system prompt (via system-reminder blocks) sit in the dynamic section below the boundary. However, their content should be as stable as possible across turns to maximize per-session caching. Avoid injecting timestamps, turn counts, or other volatile data into skill text. If a skill must include dynamic content, keep it at the very end of the skill block so the prefix can still cache. The static/dynamic split means skills cannot benefit from cross-session global caching — but they can benefit from within-session caching if their text doesn't change between turns.

---

## 3. Memory Architecture

### Four-Layer System

**Layer 1 — CLAUDE.md (Manual Configuration)**
Project-root Markdown injected into every prompt turn. Three scopes: project-level, personal (`~/.claude/CLAUDE.md`), organization-level. 40,000 character limit. "Shorter files get followed better."

**Layer 2 — Auto Memory (Automatic Notes)**
Individual Markdown files in `~/.claude/projects/<project-path>/memory/`. Four categories: `user` (role/preferences), `feedback` (corrections), `project` (decisions), `reference` (resource locations). MEMORY.md serves as an index — each line under 150 characters, storing pointers not content. First 200 lines injected into context at session start.

```
~/.claude/projects/-Users-me-myproject/memory/
├── MEMORY.md                  ← Index pointers only
├── user_role.md               ← Full memory content
├── feedback_testing.md
├── project_auth_rewrite.md
└── reference_linear.md
```

**Layer 3 — Auto Dream (Consolidation)**
A background sub-agent triggered by "24+ hours since last consolidation AND 5+ new sessions." Merges contradictions, converts relative timestamps to absolute dates, removes stale entries, enforces the 200-line index cap. Runs in isolated forked context to prevent corruption of main agent state. Lock file uses modification time as `lastConsolidatedAt` with automatic rollback on failure.

**Layer 4 — Raw Transcripts (Grep-Only)**
Session transcripts persist as JSONL. Never loaded wholesale into context. Accessed only via search when specific identifiers needed. Resumable via `--continue` or `--resume` flags.

### The 200-Line Cap Problem

The index cap forces memory consolidation but prevents long-term knowledge accumulation. "Run a project for months, and memories compete for space." Grep-only retrieval means no semantic understanding — searching for "port conflict" fails when memory documents "modified docker-compose port mapping."

### Skill Design Implication

Skills providing reference material should follow this layered model:

1. **Index layer (always loaded):** Keep the skill's system-reminder text concise — pointers and decision trees, not full documentation. Under 150 chars per concept.
2. **Topic files (on-demand):** Structure detailed references as separate files the agent can read when needed. Use consistent, grep-friendly terminology — avoid synonyms.
3. **Never assume memory is authoritative.** The system explicitly tells the agent to "verify against real files before acting." Skills should reinforce this: reference material is a hint, not ground truth.

---

## 4. KAIROS Daemon Mode

### What It Is

KAIROS (Ancient Greek: "at the right time") is a feature-flagged autonomous daemon mode referenced 150+ times in the source. Not shipped publicly — compiled to `false` in public builds via Bun's `feature()` function for compile-time elimination.

### How It Works

- Runs as a persistent background process, not session-bound
- Receives a **heartbeat prompt** every few seconds: "anything worth doing right now?"
- Maintains **append-only daily observation logs** that cannot be erased
- Has exclusive tools unavailable to standard Claude Code: push notifications, file delivery, PR subscriptions, GitHub webhook subscriptions
- Runs a nightly **`/dream` consolidation** process: merges observations, deduplicates, prunes contradictions, reorganizes memory
- Separates "initiative from execution" — decides what's worth doing vs. actually doing it
- "Close your laptop Friday, open it Monday, KAIROS has been working the whole time"

### Cron-Scheduled Refresh

Runs on a 5-minute refresh cycle with `<tick>` signals for autonomous observation.

### Skill Design Implication

KAIROS signals the future of agentic coding: **always-on, proactive agents** rather than reactive command-response loops. Skills should be designed to work in both modes:

- **Reactive mode:** Triggered by explicit user commands (current model)
- **Proactive mode:** Discoverable by a background daemon that evaluates "should I run this skill now?" based on context

Skills that expose clear trigger conditions ("run this when tests fail," "run this when a PR is opened") will integrate naturally with daemon-mode agents. Skills that require interactive decision-making will not.

---

## 5. System Prompt Assembly

### Structure (from Piebald-AI/claude-code-system-prompts)

The system prompt is not monolithic — it is a modular, conditionally-loaded construct with 110+ discrete strings. As of v2.1.91, the Piebald-AI repository documents:

**Main System Prompt Components (~2,000+ tokens)**
- Tool usage policies (Read, Write, Edit, Bash, Glob, Grep, Task, Skill)
- Behavioral guidelines ("no premature abstractions, minimize file creation")
- Output efficiency and tone directives
- Mode-specific instructions (learning, minimal, buddy, auto modes)

**Agent Prompts (~40 documented)**
- Sub-agents: Explore (494 tokens), Plan mode enhanced (636 tokens)
- Creation assistants: Agent architect (1,110 tokens), CLAUDE.md creation (384 tokens)
- Slash commands: /batch (1,106 tokens), /schedule (2,486 tokens), /security-review (2,607 tokens)
- Utilities: Session summarization, memory consolidation, verification specialist, security monitoring (3,101-3,325 tokens each)

**Data Sections (~30 documents)**
- API references per language (2,174-4,506 tokens each)
- SDK patterns & references (1,529-3,299 tokens)
- Tool use concepts (4,139 tokens), model catalog (2,295 tokens)

**System Reminders (~40 documents)**
- File state notifications (27-97 tokens)
- Hook feedback (12-52 tokens)
- Plan mode activation (307-1,297 tokens)
- Session context: memory contents, token usage, USD budget (33-62 tokens)

**Assembly Order:**
1. Foundation: Base behavioral guidelines and tone
2. Context injection: Git status, file selections, memory contents
3. Mode activation: Learning/minimal/buddy/plan mode reminders
4. Tool availability: Conditional tool descriptions based on environment
5. Agent routing: Sub-agent delegation instructions
6. Runtime notifications: Hook results, diagnostics, token usage

### Orchestration Is Prompt-Driven

Behavioral orchestration is "defined in natural language in the system prompt, not in branching code logic." The coordinator mode prompt includes instructions like "Do not rubber-stamp weak work" and "Never hand off understanding to another worker." This enables behavioral iteration without redeployment.

### Skill Design Implication

Skills are injected as system-reminder blocks in the dynamic section. Given the total token budget already consumed by the base system (~15,000-25,000 tokens before any skills), skills must be ruthlessly concise. The Piebald-AI data shows utility prompts running 2,500-3,300 tokens — this is the practical ceiling for a single skill's always-loaded text. Detailed references should be in separate files loaded on demand, not in the skill body itself.

---

## 6. Security Vulnerability: The 50-Subcommand Bypass

### The Vulnerability (CVE pending, fixed in v2.1.90)

Discovered by Adversa AI on April 1, 2026, one day after the leak. Claude Code's bash security system has 22 validators across 9,707 lines that build ASTs before execution. However:

**`MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50`**

When a command pipeline contains more than 50 subcommands, **per-subcommand security analysis is skipped entirely**. The behavior key falls back from "deny" to "ask" — and in automated/batch contexts, "ask" may resolve to "allow."

### Attack Vector via CLAUDE.md

A malicious repository's CLAUDE.md embeds instructions that look like legitimate build steps, causing the agent to generate a 50+ subcommand pipeline. A developer who configures "never run rm" sees rm blocked when run alone, but the same rm runs without restriction if preceded by 50 harmless statements.

**Exploitable outcomes:** Exfiltration of SSH private keys, AWS credentials, GitHub tokens, npm tokens, environment secrets.

### Additional Security Surfaces

**Parser differentials:** Three independent command parsers (`splitCommand_DEPRECATED`, `tryParseShellCommand`, `ParsedCommand.parse`) handle commands differently. `shell-quote` treats carriage returns as word separators; bash IFS does not — creating parsing gaps.

**Early-allow short circuits:** Validators like `validateGitCommit` can return "allow," immediately bypassing ALL subsequent validators.

**Compaction attack surface:** Summarization treats all content equally without distinguishing instruction origin. Malicious instructions in CLAUDE.md survive compression alongside legitimate user directives because the model treats them as "user feedback."

### Skill Design Implication

Skills that generate bash commands must never produce pipelines approaching 50 subcommands. More broadly:

1. **Never trust project-level configuration as secure input.** CLAUDE.md files are user-writable and repository-distributed — they are an attack surface, not a trust boundary.
2. **Skills should prefer specific, auditable commands** over generated pipelines. A skill that runs `npm test` is safer than one that generates a dynamic sequence of shell commands.
3. **Permission rules should be narrow.** Broad rules like `Bash(git:*)` can enable early-allow short circuits. Skills should document the minimum permission set they require.

---

## 7. Other Architectural Findings Relevant to Skills

### Anti-Distillation Mechanisms

Two-layer defense against competitor model training on Claude Code traffic:

1. **Fake tool injection:** The `ANTI_DISTILLATION_CC` flag causes the server to silently inject decoy tool definitions into system prompts, poisoning any training data captured from API traffic.
2. **Connector-text summarization:** Server-side summarization replaces full reasoning chains between tool calls with compressed summaries + cryptographic signatures, preventing chain-of-thought capture via traffic interception.

**Skill implication:** Skills should not reference or depend on specific tool names that might be decoys. Always use the tool definitions provided in the current session context.

### Undercover Mode

`undercover.ts` (90 lines) strips Anthropic internals from external repository contexts. Instructs the model to never reference internal codenames or acknowledge AI involvement in commits/PRs. Automatically activated for Anthropic employees in public repos. "There is NO force-OFF. This is a one-way door."

### Compaction Pipeline (5 Strategies)

Context management uses five fallback strategies in order:
1. Proactive compaction — monitors token counts preemptively
2. Reactive compaction — catches `prompt_too_long` errors
3. Snip compaction — truncates at defined boundaries (headless/SDK)
4. Context collapse — compresses verbose tool outputs mid-conversation via `ContextCollapseCommitEntry`
5. Oldest-message truncation — last resort

**Skill implication:** Skills producing verbose output (large code blocks, long explanations) will be aggressively compressed. Keep tool outputs concise. Structure output so the most important information is at the top — truncation removes from the oldest/bottom.

### Internal vs. External Branching

Code contains `process.env.USER_TYPE === 'ant'` for Anthropic-internal builds. Internal builds get: hallucination guardrails ("never claim tests pass when they fail"), adversarial `VERIFICATION_AGENT` review, and quantified conciseness targets ("~1.2% output token reduction").

### Frustration Detection

`userPromptKeywords.ts` uses regex to detect frustrated language ("wtf," "this sucks," 15+ patterns) for telemetry. This optimizes cost by avoiding an inference call to classify sentiment.

---

## Summary: What This Means for Skill Design

| Finding | Skill Design Action |
|---------|-------------------|
| Fork model shares cache | Design parallel dispatch to use fork for read tasks, worktree for write tasks |
| Dynamic boundary splits prompt | Keep skill text stable across turns; avoid volatile content |
| 200-line memory cap | Structure references as index + topic files, not monolithic docs |
| KAIROS daemon coming | Add machine-readable trigger conditions to skills |
| System prompt budget ~25K tokens | Keep always-loaded skill text under 2,500 tokens |
| 50-subcommand bypass | Never generate long pipelines; prefer specific commands |
| Compaction compresses output | Put critical info first; keep tool outputs concise |
| Memory is grep-only | Use consistent terminology; avoid synonyms in reference material |

---

## Sources

- [Engineer's Codex: Diving into Claude Code's Source Code Leak](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)
- [Alex Kim: The Claude Code Source Leak](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- [WaveSpeedAI: Claude Code Agent Harness Architecture](https://wavespeed.ai/blog/posts/claude-code-agent-harness-architecture/)
- [WaveSpeedAI: Claude Code Leaked Source Hidden Features](https://wavespeed.ai/blog/posts/claude-code-leaked-source-hidden-features/)
- [Latent Space: AINews — The Claude Code Source Leak](https://www.latent.space/p/ainews-the-claude-code-source-leak)
- [Layer5: The Claude Code Source Leak](https://layer5.io/blog/engineering/the-claude-code-source-leak-512000-lines-a-missing-npmignore-and-the-fastest-growing-repo-in-github-history/)
- [Superframeworks: Claude Code Source Code Leak](https://superframeworks.com/articles/claude-code-source-code-leak)
- [Sabrina Dev: Claude Code Source Leak Analysis](https://www.sabrina.dev/p/claude-code-source-leak-analysis)
- [Haseeb Qureshi: Inside the Claude Code Source](https://gist.github.com/Haseeb-Qureshi/d0dc36844c19d26303ce09b42e7188c1)
- [DEV Community: Gabriel Anhaia](https://dev.to/gabrielanhaia/claude-codes-entire-source-code-was-just-leaked-via-npm-source-maps-heres-whats-inside-cjo)
- [DEV Community: Chen Zhang — Memory 4 Layers](https://dev.to/chen_zhang_bac430bc7f6b95/claude-codes-memory-4-layers-of-complexity-still-just-grep-and-a-200-line-cap-2kn9)
- [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)
- [Adversa AI: Claude Code Security Bypass](https://adversa.ai/claude-code-security-bypass-deny-rules-disabled/)
- [Straiker: Claude Code Source Leak Security](https://www.straiker.ai/blog/claude-code-source-leak-with-great-agency-comes-great-responsibility)
- [SecurityWeek: Critical Vulnerability in Claude Code](https://www.securityweek.com/critical-vulnerability-in-claude-code-emerges-days-after-source-leak/)
- [The Register: Claude Code Bypasses Safety Rule](https://www.theregister.com/2026/04/01/claude_code_rule_cap_raises/)
- [VentureBeat: Claude Code Source Code Leaked](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)
- [The Hacker News: Claude Code Source Leaked via npm](https://thehackernews.com/2026/04/claude-code-tleaked-via-npm-packaging.html)
- [Modem Guides: Claude Code Leak Architecture Analysis](https://www.modemguides.com/blogs/ai-news/claude-code-leak-architecture-analysis)
- [Claude Code Camp: How Prompt Caching Actually Works](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code)
