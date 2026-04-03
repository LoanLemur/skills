# Anthropic Official Documentation: Gaps and Corrections

**Source:** Fetched 2026-04-03 from code.claude.com/docs/en/
**Pages reviewed:** best-practices, skills, sub-agents, hooks, agent-teams, plugins, interactive-mode, permission-modes

This document covers features the synthesis either missed entirely or described incompletely.

---

## 1. CLAUDE.md `@path/to/import` Syntax

**Source:** https://code.claude.com/docs/en/best-practices

CLAUDE.md files support an `@path/to/import` syntax that pulls in other files by reference. This allows modular instruction sets without duplicating content.

```markdown
# CLAUDE.md
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

The `@` prefix works with relative paths (from the CLAUDE.md location), absolute paths, and home-directory paths (`~`). This is distinct from the `@` mention syntax in the interactive prompt (which triggers file autocomplete for context injection). The import syntax is a preprocessing step that splices file contents into the CLAUDE.md before Claude reads it.

**What the synthesis missed:** The synthesis mentions CLAUDE.md extensively but never documents this import mechanism. This is critical for monorepo setups where shared instructions live in one place but need inclusion across multiple CLAUDE.md files.

---

## 2. `/btw` -- Side Questions Without Context Growth

**Source:** https://code.claude.com/docs/en/interactive-mode

`/btw` opens a quick-answer overlay that sees the full conversation but:
- Has **no tool access** (cannot read files, run commands, or search)
- Produces a **single response** (no follow-up turns)
- The question and answer are **ephemeral** -- they never enter conversation history
- Can run **while Claude is working** on a main turn without interrupting it
- Reuses the parent conversation's **prompt cache**, so cost is minimal

```
/btw what was the name of that config file again?
```

Press Space, Enter, or Escape to dismiss the overlay.

The docs explicitly contrast `/btw` with subagents: "/btw is the inverse of a subagent: it sees your full conversation but has no tools, while a subagent has full tools but starts with an empty context."

**What the synthesis missed:** Not mentioned at all. This is a practical context-management feature that prevents the common anti-pattern of asking small clarifying questions that pollute the main context window.

---

## 3. `/rewind` and Checkpointing

**Source:** https://code.claude.com/docs/en/best-practices, https://code.claude.com/docs/en/interactive-mode

Claude automatically creates checkpoints before every change. Two ways to access:

- **Double-tap Escape** (`Esc + Esc`): opens the rewind menu
- **`/rewind` command**: same menu

From the rewind menu you can:
1. **Restore conversation only** -- roll back the chat but keep code changes
2. **Restore code only** -- revert files but keep conversation history
3. **Restore both** -- full rollback to that checkpoint
4. **Summarize from here** -- condense messages from that point forward while keeping earlier context intact (selective compaction)

Key properties:
- Checkpoints persist across sessions (close terminal, rewind later)
- Only tracks changes made **by Claude**, not external processes
- Not a replacement for git

The "Summarize from here" option is particularly notable -- it's a surgical compaction tool that lets you keep early context intact while condensing a verbose middle section.

**What the synthesis missed:** The synthesis mentions `/clear` and compaction but never documents `/rewind`, checkpointing, or the selective summarization capability. The synthesis's P7 (Context Is the Fundamental Resource) should have included this as a primary context-management mechanism.

---

## 4. Auto Mode Classifier

**Source:** https://code.claude.com/docs/en/permission-modes

Auto mode is a permission mode where a **separate classifier model** reviews every action before it executes. This is not just "skip all permissions" -- it's a layered safety system.

### Architecture

- **Classifier model:** Always Claude Sonnet 4.6, regardless of main session model
- **Input to classifier:** User messages and tool calls only. Claude's own text and tool results are **stripped out** -- this means injected instructions in files/web pages cannot manipulate the classifier
- **Additional defense:** A server-side probe scans incoming tool results and flags suspicious content before Claude reads it
- **CLAUDE.md is included:** The classifier sees your project instructions, so project-specific actions described there are factored into decisions
- **Cost:** Classifier calls count toward token usage. Mainly triggered by shell commands and network operations (read-only actions and file edits don't trigger it)

### Decision order (first match wins)

1. Actions matching explicit allow/deny rules resolve immediately
2. Read-only actions and file edits in working directory are auto-approved (except protected directories)
3. Everything else goes to the classifier
4. If classifier blocks, Claude receives the reason and tries an alternative

### What it blocks by default

Blocked:
- `curl | bash` or executing downloaded scripts
- Sending sensitive data to external endpoints
- Production deploys and migrations
- Mass deletion on cloud storage
- Granting IAM or repo permissions
- Force push or pushing directly to `main`
- Irreversibly destroying pre-existing files

Allowed:
- Local file operations in working directory
- Installing dependencies from lock files
- Reading `.env` and sending credentials to their matching API
- Read-only HTTP requests
- Pushing to the current branch or one Claude created

### Fallback thresholds

- **3 consecutive blocks** or **20 total blocks** in one session: auto mode pauses, resumes interactive prompting
- In non-interactive mode (`-p` flag): aborts the session entirely
- Thresholds are **not configurable**
- Approving a prompted action resets the denial counters

### Auto mode + subagents

- Classifier evaluates the delegated task **at spawn time**
- Inside the subagent, auto mode runs with the same rules as the parent
- Any `permissionMode` in the subagent's frontmatter is **ignored** under auto mode
- When subagent finishes, classifier **reviews its full action history** for post-hoc compromise detection

### Entry into auto mode drops dangerous allow rules

On entering auto mode, Claude Code drops allow rules that grant arbitrary code execution: blanket `Bash(*)`, wildcarded script interpreters like `Bash(python*)` or `Bash(node*)`, package-manager run commands, and any `Agent` allow rule. Narrow rules like `Bash(npm test)` are preserved. Dropped rules are restored when leaving auto mode.

### Configuration

- Available on Team, Enterprise, and API plans only
- Requires admin enablement for Team/Enterprise
- Run `claude auto-mode defaults` to see the full default rule lists
- Administrators can configure trusted infrastructure via `autoMode.environment` in managed settings

**What the synthesis missed:** The synthesis mentions auto mode briefly in the tool landscape (01) but treats it as a simple "skip permissions" toggle. The actual implementation is a sophisticated multi-layer system with a separate classifier model, prompt-injection defenses, post-hoc subagent review, and configurable trusted infrastructure. This is arguably the most important safety feature in Claude Code for autonomous workflows.

---

## 5. Custom Subagents (`.claude/agents/`)

**Source:** https://code.claude.com/docs/en/sub-agents

### File format

Markdown files with YAML frontmatter, stored in `.claude/agents/` (project) or `~/.claude/agents/` (user):

```markdown
# .claude/agents/security-reviewer.md
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure data handling
```

### All frontmatter fields

| Field | Required | Description |
|:------|:---------|:------------|
| `name` | Yes | Unique identifier (lowercase + hyphens) |
| `description` | Yes | When Claude should delegate to this subagent |
| `tools` | No | Allowlist of tools. Inherits all if omitted |
| `disallowedTools` | No | Denylist, removed from inherited set |
| `model` | No | `sonnet`, `opus`, `haiku`, full model ID, or `inherit` (default) |
| `permissionMode` | No | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | Maximum agentic turns before stopping |
| `skills` | No | Skills to preload into context at startup |
| `mcpServers` | No | MCP servers (inline definitions or references) |
| `hooks` | No | Lifecycle hooks scoped to this subagent |
| `memory` | No | Persistent memory scope: `user`, `project`, or `local` |
| `background` | No | `true` to always run as background task |
| `effort` | No | `low`, `medium`, `high`, `max` (Opus 4.6 only) |
| `isolation` | No | `worktree` for temporary git worktree isolation |
| `color` | No | Display color: red, blue, green, yellow, purple, orange, pink, cyan |
| `initialPrompt` | No | Auto-submitted first user turn when running as `--agent` |

### Scope and priority (highest wins)

1. Managed settings (organization-wide)
2. `--agents` CLI flag (JSON, session-only)
3. `.claude/agents/` (project)
4. `~/.claude/agents/` (user/personal)
5. Plugin `agents/` directory

### Built-in subagents

| Agent | Model | Purpose |
|:------|:------|:--------|
| **Explore** | Haiku | Fast, read-only codebase search. Supports `quick`, `medium`, `very thorough` levels |
| **Plan** | Inherits | Read-only research during plan mode |
| **general-purpose** | Inherits | Complex multi-step tasks requiring both exploration and action |
| **statusline-setup** | Sonnet | Configuring the status line |
| **Claude Code Guide** | Haiku | Answering questions about Claude Code features |

### Key capabilities

- **Persistent memory:** `memory: user` gives the subagent a directory at `~/.claude/agent-memory/<name>/` that persists across sessions. The first 200 lines / 25KB of `MEMORY.md` are auto-included in the system prompt.
- **MCP server scoping:** Inline MCP definitions connect only when the subagent starts and disconnect when it finishes. This keeps MCP tool descriptions out of the main context.
- **Tool restriction with `Agent(type)` syntax:** When running as `--agent`, you can restrict which subagent types can be spawned: `tools: Agent(worker, researcher), Read, Bash`
- **Background vs foreground:** Background subagents run concurrently. Permissions are pre-approved at launch; anything not pre-approved is auto-denied. Press `Ctrl+B` to background a running task.
- **`@` mention syntax:** Type `@agent-name` to force delegation to a specific subagent for one task.
- **Session-wide agent:** `claude --agent code-reviewer` makes the entire session use that subagent's system prompt, tools, and model. Can be set as default via `"agent": "code-reviewer"` in `.claude/settings.json`.
- **`/agents` command:** Interactive UI for creating, editing, and managing subagents. Supports Claude-generated definitions.

### Subagents cannot spawn other subagents

This is an explicit architectural constraint. For nested delegation, use skills or chain subagents from the main conversation.

**What the synthesis missed:** The synthesis mentions subagents as a context-management tool but doesn't document: the file format, any frontmatter fields, persistent memory, MCP scoping, tool restriction syntax, the `--agent` flag for session-wide agent mode, or the `/agents` interactive management command. The synthesis also doesn't mention that subagent definitions double as agent team teammate definitions.

---

## 6. Skills: Full Specification

**Source:** https://code.claude.com/docs/en/skills

### `disable-model-invocation: true`

This frontmatter field prevents Claude from automatically loading the skill. The skill is only triggered when a user explicitly types `/skill-name`. Use for workflows with side effects (deploy, commit, send-slack-message) where you don't want Claude deciding on its own to invoke them.

When set, the skill description is **not loaded into context** at all, saving context budget.

The inverse is `user-invocable: false`, which hides the skill from the `/` menu but lets Claude invoke it automatically. Use for background knowledge that isn't actionable as a command.

| Frontmatter | User can invoke | Claude can invoke | Context loading |
|:------------|:---------------|:-----------------|:----------------|
| (default) | Yes | Yes | Description always in context |
| `disable-model-invocation: true` | Yes | No | Not in context |
| `user-invocable: false` | No | Yes | Description always in context |

### All frontmatter fields

| Field | Required | Description |
|:------|:---------|:------------|
| `name` | No | Display name (defaults to directory name). Lowercase, hyphens, max 64 chars |
| `description` | Recommended | What the skill does. Front-load key use case; truncated at 250 chars |
| `argument-hint` | No | Hint shown during autocomplete, e.g. `[issue-number]` |
| `disable-model-invocation` | No | `true` to prevent automatic invocation |
| `user-invocable` | No | `false` to hide from `/` menu |
| `allowed-tools` | No | Tools permitted without asking when skill is active |
| `model` | No | Model to use when skill is active |
| `effort` | No | `low`, `medium`, `high`, `max` (Opus 4.6 only) |
| `context` | No | `fork` to run in a forked subagent context |
| `agent` | No | Which subagent type when `context: fork` (default: `general-purpose`) |
| `hooks` | No | Hooks scoped to this skill's lifecycle |
| `paths` | No | Glob patterns limiting when skill activates |
| `shell` | No | `bash` (default) or `powershell` |

### String substitutions in skill content

| Variable | Description |
|:---------|:------------|
| `$ARGUMENTS` | All arguments passed when invoking |
| `$ARGUMENTS[N]` or `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing the SKILL.md |

### Dynamic context injection: `` !`command` `` syntax

Shell commands in backticks prefixed with `!` execute as preprocessing before Claude sees the skill content. Output replaces the placeholder:

```yaml
---
name: pr-summary
context: fork
agent: Explore
---
## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
```

### Skills + subagents relationship

| Approach | System prompt | Task | Also loads |
|:---------|:-------------|:-----|:-----------|
| Skill with `context: fork` | From agent type | SKILL.md content | CLAUDE.md |
| Subagent with `skills` field | Subagent's markdown body | Claude's delegation message | Preloaded skills + CLAUDE.md |

### Bundled skills (ship with Claude Code)

| Skill | Purpose |
|:------|:--------|
| `/batch <instruction>` | Parallel codebase changes across git worktrees (5-30 units) |
| `/claude-api` | Load Claude API reference for your language |
| `/debug [description]` | Enable debug logging and troubleshoot |
| `/loop [interval] <prompt>` | Run a prompt repeatedly on interval |
| `/simplify [focus]` | Review changed files with 3 parallel review agents |

### Skill description budget

All skill names are always included in context, but descriptions are shortened to fit a character budget of 1% of context window (fallback: 8,000 chars). Each entry is capped at 250 characters regardless. Override with `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var.

### Agent Skills open standard

Claude Code skills follow the [Agent Skills](https://agentskills.io) open standard, which works across multiple AI tools. Claude Code extends it with invocation control, subagent execution, and dynamic context injection.

**What the synthesis missed:** The synthesis documents skills at a high level but misses: `disable-model-invocation`, `user-invocable`, the `paths` field for glob-based activation, the `context: fork` mechanism, the `!`backtick`` preprocessing syntax, the `$ARGUMENTS[N]` indexed access, the `${CLAUDE_SKILL_DIR}` variable, the description budget system, the Agent Skills open standard, bundled skills, and the skill-subagent relationship table.

---

## 7. Plugin Ecosystem

**Source:** https://code.claude.com/docs/en/plugins

Plugins bundle skills, agents, hooks, MCP servers, LSP servers, and settings into a distributable unit.

### Directory structure

```
my-plugin/
  .claude-plugin/
    plugin.json          # Manifest (required)
  skills/                # Skill directories with SKILL.md
  agents/                # Subagent markdown files
  hooks/
    hooks.json           # Hook configuration
  commands/              # Legacy command files
  .mcp.json              # MCP server configs
  .lsp.json              # LSP server configs
  bin/                   # Executables added to Bash PATH
  settings.json          # Default settings (currently only `agent` key)
```

### Plugin manifest (`.claude-plugin/plugin.json`)

```json
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "version": "1.0.0",
  "author": { "name": "Your Name" }
}
```

Additional fields: `homepage`, `repository`, `license`.

### Key properties

- Skills are namespaced: `/plugin-name:skill-name` (prevents conflicts)
- `/plugin` command to browse the marketplace
- `--plugin-dir ./my-plugin` for local development/testing
- `/reload-plugins` to pick up changes without restarting
- Plugin subagents **cannot use** `hooks`, `mcpServers`, or `permissionMode` frontmatter (security restriction)
- LSP servers (`.lsp.json`) give Claude real-time code intelligence
- `bin/` directory executables are added to PATH while plugin is enabled
- `settings.json` can set a default agent for the session
- Official marketplace submission via claude.ai/settings/plugins/submit

### Plugin vs standalone

| Standalone (`.claude/`) | Plugin |
|:------------------------|:-------|
| Only this project | Shareable via marketplaces |
| Short names: `/deploy` | Namespaced: `/my-plugin:deploy` |
| Manual copy to share | Install with `/plugin install` |

**What the synthesis missed:** Plugins are not mentioned in the synthesis at all. This is an entire distribution and extension ecosystem including marketplace discovery, namespaced skills, LSP servers for code intelligence, binary distribution via `bin/`, and organizational deployment through managed settings.

---

## 8. Hooks: All 26 Events and 4 Handler Types

**Source:** https://code.claude.com/docs/en/hooks

The synthesis correctly identifies hooks as deterministic enforcement (P4), but the actual hook system is far more extensive than documented.

### Handler types (4, not 3)

1. **`command`** -- Execute shell commands. Receive JSON on stdin, exit codes control behavior.
2. **`http`** -- POST JSON to a URL. Supports headers with env var substitution.
3. **`prompt`** -- Single-turn evaluation by a Claude model. Returns yes/no as JSON.
4. **`agent`** -- Spawn a subagent with tool access (Read, Grep, Glob) to verify conditions.

### All 26 hook events

| Event | Matcher | Description |
|:------|:--------|:------------|
| `SessionStart` | Initiation type (`startup`, `resume`, `clear`, `compact`) | Session begins or resumes |
| `SessionEnd` | End reason (`clear`, `resume`, `logout`, `prompt_input_exit`) | Session terminates |
| `InstructionsLoaded` | Load reason (`session_start`, `nested_traversal`, `path_glob_match`) | CLAUDE.md or rules files loaded |
| `UserPromptSubmit` | None | User submits a prompt |
| `PreToolUse` | Tool name | Before a tool call executes |
| `PostToolUse` | Tool name | After a tool completes successfully |
| `PostToolUseFailure` | Tool name | When a tool execution fails |
| `PermissionRequest` | Tool name | Permission dialog appears |
| `PermissionDenied` | Tool name | Auto mode classifier denies a tool call |
| `Notification` | Type (`permission_prompt`, `idle_prompt`, `auth_success`) | Claude Code sends a notification |
| `SubagentStart` | Agent type | Subagent is spawned |
| `SubagentStop` | Agent type | Subagent finishes |
| `TaskCreated` | None | Task being created via TaskCreate |
| `TaskCompleted` | None | Task being marked complete |
| `TeammateIdle` | None | Agent team teammate about to go idle |
| `Stop` | None | Claude finishes responding |
| `StopFailure` | Error type (`rate_limit`, `authentication_failed`, `billing_error`) | Turn ends due to API error |
| `ConfigChange` | Config source (`user_settings`, `project_settings`, `local_settings`) | Configuration file changes |
| `CwdChanged` | None | Working directory changes |
| `FileChanged` | Filename (basename) | Watched file changes on disk |
| `WorktreeCreate` | None | Worktree being created |
| `WorktreeRemove` | None | Worktree being removed |
| `PreCompact` | Trigger (`manual`, `auto`) | Before context compaction |
| `PostCompact` | Trigger (`manual`, `auto`) | After compaction completes |
| `Elicitation` | MCP server name | MCP server requests user input |
| `ElicitationResult` | MCP server name | User responds to MCP elicitation |

### Exit code behavior

| Code | Meaning | Effect |
|:-----|:--------|:-------|
| 0 | Success | Parse stdout for JSON output |
| 2 | Blocking error | Block the action; stderr as error message |
| Other | Non-blocking error | Log stderr in verbose mode; continue |

### Hook-specific output (JSON on stdout)

Hooks can return structured JSON to influence Claude's behavior:
- `decision`: `allow`, `deny`, `ask`, `defer` (for permission hooks)
- `additionalContext`: injected into Claude's context
- `updatedInput`: modify the tool input before execution
- `suppressOutput`: hide the action from the user
- `systemMessage`: warning displayed to the user

### Configuration locations

- `~/.claude/settings.json` (all projects)
- `.claude/settings.json` (project, shareable)
- `.claude/settings.local.json` (project, gitignored)
- Managed policy settings (organization-wide)
- Plugin `hooks/hooks.json` (bundled with plugin)
- Skill/Agent frontmatter (scoped to component lifecycle)

### Environment variables for hooks

| Variable | Scope | Purpose |
|:---------|:------|:--------|
| `$CLAUDE_PROJECT_DIR` | All hooks | Project root path |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin hooks | Plugin installation directory |
| `${CLAUDE_PLUGIN_DATA}` | Plugin hooks | Plugin persistent data directory |
| `$CLAUDE_ENV_FILE` | SessionStart, CwdChanged, FileChanged | Persist environment variables |
| `$CLAUDE_CODE_REMOTE` | All hooks | `true` in web environments |

**What the synthesis missed:** The synthesis treats hooks as a simple "run script on event" mechanism. It misses: the 4 handler types (especially `prompt` and `agent` which are AI-powered), the full event list (26 events covering session lifecycle, compaction, worktrees, MCP elicitation, file watching, teammate coordination), the structured JSON output schema with decision control, the `updatedInput` capability to modify tool inputs, and the `if` field for permission-rule-style filtering within hooks.

---

## 9. Agent Teams

**Source:** https://code.claude.com/docs/en/agent-teams

Agent teams are **experimental** (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`).

### How they differ from subagents

| | Subagents | Agent teams |
|:--|:----------|:------------|
| **Context** | Own window; results return to caller | Own window; fully independent |
| **Communication** | Report back to main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-coordination |
| **Best for** | Focused tasks where only result matters | Complex work requiring discussion |
| **Token cost** | Lower (results summarized) | Higher (each is a separate instance) |

### Architecture

- **Team lead:** The main session that creates and coordinates the team
- **Teammates:** Separate Claude Code instances with independent context
- **Task list:** Shared work items with three states: pending, in progress, completed. Supports dependencies.
- **Mailbox:** Messaging system for inter-agent communication

### Display modes

- **In-process:** All teammates in main terminal. `Shift+Down` to cycle. Works everywhere.
- **Split panes:** Each teammate in its own tmux/iTerm2 pane. Click to interact.

### Task coordination

- Tasks have dependency tracking -- blocked tasks auto-unblock when dependencies complete
- File locking prevents race conditions on simultaneous task claims
- Teammates can self-claim tasks after finishing one
- Lead can assign tasks explicitly

### Quality gates via hooks

- `TeammateIdle`: Runs when teammate is about to idle. Exit code 2 sends feedback and keeps them working.
- `TaskCreated`: Runs on task creation. Exit code 2 prevents creation with feedback.
- `TaskCompleted`: Runs on task completion. Exit code 2 prevents completion with feedback.

### Plan approval for teammates

Teammates can be required to plan before implementing. The lead reviews and approves/rejects plans. Rejected teammates revise and resubmit.

### Practical limits

- No session resumption with in-process teammates
- One team per session
- No nested teams (teammates can't spawn teams)
- Lead is fixed for the team's lifetime
- All teammates start with lead's permission mode
- Storage: `~/.claude/teams/{team-name}/config.json` and `~/.claude/tasks/{team-name}/`

### Subagent definitions as teammate roles

When spawning a teammate, you can reference any subagent type. The teammate inherits that subagent's system prompt, tools, and model.

**What the synthesis missed:** Agent teams are not mentioned in the synthesis at all. This is a complete multi-agent coordination system with shared task lists, inter-agent messaging, dependency tracking, quality gates, and plan approval workflows.

---

## 10. Additional Features Missed

### Extended thinking in skills
Including the word "ultrathink" anywhere in skill content enables extended thinking for that skill invocation.

### Skill `paths` field
Glob patterns that limit when a skill is automatically activated. When set, Claude only loads the skill when working with files matching the patterns.

### `--add-dir` and skills exception
While `--add-dir` generally grants file access only (not configuration), skills in `.claude/skills/` within an added directory ARE loaded and support live change detection.

### Skill description budget system
Descriptions are allocated 1% of context window (fallback 8K chars), each capped at 250 chars. Override via `SLASH_COMMAND_TOOL_CHAR_BUDGET`.

### `/agents` interactive command
Full CRUD interface for subagents with Claude-assisted generation. Can create subagents through guided setup or by describing what you want.

### Subagent persistent memory
Subagents can accumulate knowledge across sessions via `memory: user|project|local`. Memory directory includes `MEMORY.md` auto-loaded into the system prompt (first 200 lines / 25KB).

### Subagent `isolation: worktree`
Runs the subagent in a temporary git worktree for complete repository isolation. Auto-cleaned if no changes made.

### `claude --agent <name>`
Makes the entire session use a subagent's configuration. The subagent's system prompt **replaces** the default Claude Code system prompt entirely. CLAUDE.md still loads normally.

### Plugin LSP servers
Plugins can bundle Language Server Protocol configurations (`.lsp.json`) giving Claude real-time code intelligence for any language.

### Plugin `bin/` directory
Executables in a plugin's `bin/` directory are added to the Bash tool's PATH while the plugin is enabled.

### Bundled `/batch` skill
Orchestrates large-scale parallel changes: researches codebase, decomposes into 5-30 independent units, spawns one background agent per unit in an isolated git worktree, each implements its unit and opens a PR.

### `context: fork` for skills
Skills can run in a forked subagent context by setting `context: fork` and optionally `agent: Explore|Plan|general-purpose|<custom>`. The skill content becomes the subagent's task prompt.

---

## Summary of Synthesis Gaps

| Feature | Synthesis Status | Importance |
|:--------|:----------------|:-----------|
| Auto mode classifier (layered safety system) | Mentioned superficially | Critical -- primary safety mechanism for autonomous work |
| Agent teams | Not mentioned | High -- complete multi-agent coordination system |
| Plugin ecosystem | Not mentioned | High -- distribution/sharing mechanism |
| Subagent file format and all fields | Not documented | High -- needed to author subagents |
| Skill full frontmatter spec | Partially documented | High -- needed to author skills |
| `@import` syntax in CLAUDE.md | Not mentioned | Medium -- enables modular instructions |
| `/btw` side questions | Not mentioned | Medium -- context management tool |
| `/rewind` and checkpointing | Not mentioned | Medium -- primary undo/recovery mechanism |
| Hook event list (26 events) | Not documented | Medium -- needed for hook authoring |
| Hook handler types (4 types) | Undercounted | Medium -- `prompt` and `agent` types are AI-powered |
| Persistent subagent memory | Not mentioned | Medium -- cross-session learning |
| `disable-model-invocation` | Not mentioned | Medium -- controls skill auto-invocation |
| Skill `!`backtick`` preprocessing | Not mentioned | Medium -- dynamic context injection |
| Bundled skills (`/batch`, `/simplify`) | Not mentioned | Low-medium -- powerful built-in capabilities |
| Plugin LSP servers | Not mentioned | Low-medium -- code intelligence |
| Agent Skills open standard | Not mentioned | Low -- interoperability detail |
