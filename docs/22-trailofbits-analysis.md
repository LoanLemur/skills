# Trail of Bits Claude Code Config Analysis

Source: https://github.com/trailofbits/claude-code-config

## What This Repo Is

A complete, opinionated setup for Claude Code used by Trail of Bits -- a world-class security research firm. It provides settings, hooks, CLAUDE.md template, slash commands, MCP server configs, and a statusline script. The README doubles as extensive documentation (~610 lines) covering philosophy, configuration rationale, and usage patterns.

This is not a toy config. It is battle-tested across security audits, development, and research by a team whose job is finding vulnerabilities in other people's code.

## Repository Structure

```
settings.json                          # Permissions, hooks, env, statusline
claude-md-template.md                  # Global CLAUDE.md for ~/.claude/CLAUDE.md
mcp-template.json                      # Context7 + Exa MCP server config
hooks/
  enforce-package-manager.sh           # Example: block npm when pnpm is used
  log-gam.sh                           # Example: audit trail for GAM mutations
commands/
  fix-issue.md                         # End-to-end GitHub issue resolution
  review-pr.md                         # Multi-agent PR review + fix
  merge-dependabot.md                  # Evaluate and merge dependabot PRs
scripts/
  statusline.sh                        # Two-line status bar (model, branch, context %, cost)
.claude/commands/trailofbits/config.md # Self-installing setup wizard
```

---

## settings.json -- Deep Analysis

### Privacy First

```json
"env": {
  "DISABLE_TELEMETRY": "1",
  "DISABLE_ERROR_REPORTING": "1",
  "CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY": "1"
}
```

They disable telemetry, error reporting, and surveys individually rather than using the umbrella `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` because the umbrella also kills auto-updates. Precision over convenience.

### Explicit Defaults

```json
"enableAllProjectMcpServers": false
```

This is already the default, but they set it explicitly. Their reasoning: a compromised repo could ship malicious MCP servers via `.mcp.json`. By pinning this to `false`, it cannot be accidentally flipped. This is a security team thinking about supply chain attacks on their own tooling.

### Permission Deny Lists -- Three Categories

**1. Destructive Bash commands:**
- `rm -rf *`, `rm -fr *` (both flag orderings)
- `sudo *`
- `mkfs *`, `dd *` (disk-level destruction)
- `wget *|bash*`, `wget *| bash*` (pipe-to-shell with/without space)
- `git push --force*`, `git push *--force*` (both positions)
- `git reset --hard*`

**2. Shell config editing:**
- `Edit(~/.bashrc)`, `Edit(~/.zshrc)`
- `Edit(~/.ssh/**)`

**3. Credential/secret reading -- the longest and most revealing section:**
- SSH/GPG keys: `~/.ssh/**`, `~/.gnupg/**`
- Cloud credentials: `~/.aws/**`, `~/.azure/**`, `~/.kube/**`, `~/.docker/config.json`
- Package registry tokens: `~/.npmrc`, `~/.npm/**`, `~/.pypirc`, `~/.gem/credentials`
- Git/GitHub auth: `~/.git-credentials`, `~/.config/gh/**`
- macOS Keychain: `~/Library/Keychains/**`
- **Crypto wallets**: metamask, electrum, exodus, phantom, solflare

The crypto wallet deny list is remarkable. It reveals that this team thinks about exfiltration vectors that most configurations would never consider. They are defending against prompt injection attacks that could read wallet data from `~/Library/Application Support/`.

### Hooks -- Deterministic Enforcement

Two `PreToolUse` hooks on `Bash`, both inline shell one-liners:

**Hook 1: Block rm -rf**
```bash
CMD=$(jq -r '.tool_input.command')
if echo "$CMD" | grep -qiE '(^|;[[:space:]]*|&&[[:space:]]*|[|][|][[:space:]]*|[|][[:space:]]*)rm[[:space:]]' \
  && echo "$CMD" | grep -qiE '(^|[[:space:]])-[a-zA-Z]*[rR]|--recursive' \
  && echo "$CMD" | grep -qiE '(^|[[:space:]])-[a-zA-Z]*[fF]|--force'; then
  echo 'BLOCKED: Use trash instead of rm -rf' >&2
  exit 2
fi
```

This is not a naive pattern match. It handles:
- Commands chained with `;`, `&&`, `||`, or piped
- Both `-rf` and `-fr` flag orderings
- Mixed flags like `-rfi` or `-Rf`
- Long-form `--recursive` and `--force`
- Case-insensitive matching

The error message is actionable: "Use trash instead of rm -rf" -- telling Claude what to do, not just what not to do.

**Hook 2: Block push to main/master**
```bash
CMD=$(jq -r '.tool_input.command')
if echo "$CMD" | grep -qE 'git[[:space:]]+push.*(main|master)'; then
  echo 'BLOCKED: Use feature branches, not direct push to main' >&2
  exit 2
fi
```

Simpler pattern because the risk surface is narrower. Again, the error message instructs.

### Key Pattern: Error Messages as Instructions

Both hooks follow the same formula: `BLOCKED: {what to do instead}`. The exit code 2 means the error text is fed back to Claude as context. So the hook is not just preventing -- it is redirecting. Claude sees the error and knows the correct alternative.

---

## claude-md-template.md -- Deep Analysis

### Overall Structure

1. **One-line header** -- "Global Development Standards"
2. **Tool preferences** -- Exa over WebSearch, proactive skill usage
3. **Philosophy** -- 10 bullet points, each bold-titled with a dash explanation
4. **Code Quality** -- Hard limits (measurable), zero warnings policy, comments policy, error handling, review order, testing
5. **Development** -- Per-language toolchain tables (Python, Node/TypeScript, Rust, Bash, GitHub Actions)
6. **Workflow** -- Pre-commit checklist, commit rules, hooks and worktrees, PR description rules

### Prose Style

**Imperative, no hedging.** Not "you should prefer" but "Prefer." Not "it's recommended to" but the bare directive. Reads like military SOPs.

**Bold key + dash explanation.** Every philosophy bullet follows this pattern:
> **No speculative features** - Don't add features, flags, or configuration unless users actively need them

The bold text is the rule name. The dash text is the operationalization. You can scan the bold text for the rule you need and read the explanation only when necessary.

**Negative framing dominates.** Most rules say what NOT to do:
- No speculative features
- No premature abstraction
- No phantom features
- Replace, don't deprecate

This is deliberate. LLMs tend toward additive behavior -- they want to add code, add abstractions, add documentation. The CLAUDE.md pushes against this tendency by making "don't add" the default.

**One sentence per idea.** No compound sentences where avoidable. No filler. Compare:
> "Each dependency is attack surface and maintenance burden."

Not: "Each dependency you add increases both the attack surface of your application and creates an additional maintenance burden that the team will need to manage going forward."

### Philosophy -- What They Prioritize

The 10 philosophy bullets reveal their worldview:

1. **No speculative features** -- YAGNI, but stronger
2. **No premature abstraction** -- Rule of three
3. **Clarity over cleverness** -- Readability > density
4. **Justify new dependencies** -- Supply chain security ("attack surface")
5. **No phantom features** -- Don't document/validate what doesn't exist
6. **Replace, don't deprecate** -- No migration paths, no backward compat shims. "Proactively flag dead code"
7. **Verify at every level** -- Automated guardrails first. "Prefer structure-aware tools (ast-grep, LSPs, compilers) over text pattern matching"
8. **Bias toward action** -- Decide and move for reversible things; ask for irreversible ones
9. **Finish the job** -- Handle edge cases you can see, but don't invent scope
10. **Agent-native by default** -- "Design so agents can achieve any outcome users can"

Number 10 is fascinating and unusual. They are building with the assumption that agents are first-class users of their systems. "Tools are atomic primitives; features are outcomes described in prompts."

### Hard Limits -- Measurable Constraints

```
1. <=100 lines/function, cyclomatic complexity <=8
2. <=5 positional params
3. 100-char line length
4. Absolute imports only -- no relative (..) paths
5. Google-style docstrings on non-trivial public APIs
```

These are numbers, not vibes. An LLM can check these mechanically. Compare to a typical CLAUDE.md that says "keep functions small" -- that's unenforceable.

### Testing Philosophy

Key quotes:
- "Test behavior, not implementation."
- "Test edges and errors, not just the happy path."
- "Mock boundaries, not logic." (only slow, non-deterministic, or external)
- "Verify tests catch failures." -- They advocate mutation testing (cargo-mutants, mutmut) and property-based testing (proptest, hypothesis).

### Language Toolchain Tables

They specify exact tools per language with "replaces" columns:

| tool | replaces |
|------|----------|
| `rg` (ripgrep) | grep |
| `fd` | find |
| `ast-grep` | - |
| `shellcheck` | - |
| `trash` | rm |
| `prek` | pre-commit |

And per-language:
- Python: `uv` (not pip/poetry), `ruff` (not black/pylint/flake8), `ty` (not mypy/pyright)
- Node: `oxlint` (not eslint), `oxfmt` (not prettier), `vitest`
- Rust: clippy with pedantic, explicit deny list in `[lints.clippy]`

The Rust clippy configuration is notably detailed -- denying `unwrap_used`, `panic`, `todo`, `dbg_macro`, `print_stdout`, `allow_attributes`, `mem_forget`, `exit`. This is a security team's Rust config: no panics in production, no cheating with `#[allow]`, no accidentally committed debug output.

### Supply Chain Security -- Embedded Throughout

Not a separate section but woven into every language:
- Python: `pip-audit`, pin exact versions (`==` not `>=`), verify hashes
- Node: `pnpm audit`, pin exact versions (no `^` or `~`), enforce 24-hour publish delay, block postinstall scripts
- Rust: `cargo deny check`
- GitHub Actions: Pin to SHA hashes with version comments, scan with `zizmor`

### Workflow Conventions

**PR descriptions:**
> "Describe what the code does now -- not discarded approaches, prior iterations, or alternatives. Only describe what's in the diff."
> "Use plain, factual language. A bug fix is a bug fix, not a 'critical stability improvement.' Avoid: critical, crucial, essential, significant, comprehensive, robust, elegant."

This is anti-LLM-speak. They are explicitly countering the model's tendency toward inflated language.

---

## Hooks -- Deep Analysis

### enforce-package-manager.sh

12 lines. Checks if `pnpm-lock.yaml` exists in the project directory, and if so, blocks any command starting with `npm`. Clean, focused, generalizable.

Key design choices:
- Uses `$CLAUDE_PROJECT_DIR` env var (set by Claude Code)
- Exits 0 (allow) on every non-matching case
- Only activates when the project actually uses pnpm

### log-gam.sh

An audit logger for Google Apps Manager mutations. This is the most sophisticated hook in the repo and reveals the pattern for building audit trails:

1. Parse command from stdin via jq
2. Check if it's a GAM command (early exit if not)
3. Classify as read or write using verb pattern lists
4. Skip reads entirely
5. For writes, extract the action verb
6. Log to JSONL with timestamp, action, command, status
7. Print a reminder to the operator

The verb lists are version-pinned: "verified against GamCommands.txt v7.33.00". They version their regex patterns.

---

## Commands -- Deep Analysis

### fix-issue.md

The most comprehensive command (~230 lines). A complete autonomous workflow:

1. **Research** -- Uses Exa for external context when needed
2. **Plan** -- Writes implementation plan to a file
3. **Create branch** -- Detects issue type for prefix (fix/, feat/, refactor/, docs/)
4. **Implement** -- Follows project CLAUDE.md
5. **Build, test, lint** -- CI-first discovery: reads `.github/workflows/` before falling back to language defaults
6. **Self-review** -- Uses `/pr-review-toolkit:review-pr` for code, manual review for docs
7. **Fix findings** -- Addresses all P1-P3, can dismiss with reasoning
8. **Commit and push** -- Deletes the plan file first
9. **Create PR** -- Links to issue
10. **Comment on issue** -- Posts summary

Key design insight: **CI is the source of truth for quality gates.** Step 5a explicitly says to read CI workflows first and only fall back to hardcoded defaults when CI is absent. This prevents the agent from running a different set of checks than what the project actually enforces.

The command also handles upstream vs origin remotes, which matters for fork-based workflows.

### review-pr.md

Multi-agent review with five parallel reviewers:
- 3 from pr-review-toolkit (code quality, silent failures, test analysis)
- 1 Codex (`gpt-5.3-codex` with `xhigh` reasoning)
- 1 Gemini (`gemini-3-pro-preview`)

Findings are deduplicated, ranked P1-P4, and the command fixes P1-P3 before pushing. The multi-model approach is deliberate: different models catch different things.

Notable: the Gemini reviewer pipes the diff via stdin to avoid shell metacharacter issues with heredocs. This is a practical battle scar turned into a standard operating procedure.

### merge-dependabot.md

The most complex command (~350+ lines). A full dependency management workflow:

1. **Phase 0**: Audit dependabot config (ecosystem coverage, uv vs pip detection, cooldown, grouping)
2. **Phase 1**: Discovery and baseline (verify main is healthy before touching anything)
3. **Phase 2**: Dependency graph analysis (transitive deps, batch overlapping PRs)
4. **Phase 3**: Parallel evaluation via subagents (library deps and actions deps get different prompts)
5. **Phase 4**: Sequential merge with post-merge re-testing
6. **Phase 5**: Cleanup and report

The turn budget management section is notable:
> "At 75% of turns used: Stop launching new evaluations. Merge any PRs already evaluated as PASS."
> "Prioritize merging over analysis."

This is resource-aware agent design -- graceful degradation when approaching limits.

### /trailofbits:config

A self-installing setup wizard. The command itself is a prompt that instructs Claude to:
1. Inventory what exists in `~/.claude/`
2. Ask the user what to install (multi-select)
3. Fetch files from GitHub raw URLs
4. Merge into existing configs (never silently overwrite CLAUDE.md)
5. Self-install to `~/.claude/commands/trailofbits/config.md`

The self-installation pattern is clever: after first use, the command lives in `~/.claude/` and works from any directory without needing the repo cloned.

---

## Statusline Script

A 150-line bash script that produces a two-line status display:

```
[Opus 4.6] folder | branch
████████⣿⣿⣿⣿ 67% | $1.42 | 8m 23s ↻89%
```

Reads JSON from stdin (Claude Code pipes session data), extracts everything in a single `jq` call for performance, with a bash-level fallback if jq crashes. Context bar is color-coded: green <50%, yellow 50-79%, red 80%+. Cache hit rate shown as `↻89%`.

Design choice: uses Braille characters (`⣿`) for the empty portion of the progress bar, which renders at the same width as the filled blocks (`█`). Attention to visual polish.

---

## Layered Defense Model

Trail of Bits uses three layers, each with different enforcement characteristics:

### Layer 1: settings.json (Deterministic, Always-On)

- Permission deny lists: hard blocks that cannot be overridden by conversation
- PreToolUse hooks: regex-based blocking that fires on every tool call
- Environment variables: telemetry/privacy controls

**What goes here:** Things that must NEVER happen, regardless of context. Credential exfiltration, destructive commands, privacy violations.

### Layer 2: CLAUDE.md (Soft, Advisory)

- Philosophy and design principles
- Code quality standards with measurable thresholds
- Language-specific toolchain preferences
- Workflow conventions

**What goes here:** Things that SHOULD happen but where context might justify exceptions. The model can override these under pressure, which is why the critical stuff is in Layer 1.

### Layer 3: Commands (Procedural, On-Demand)

- Multi-step workflows triggered by slash commands
- Encode organizational processes (review, issue fixing, dependency management)
- Can invoke other tools (subagents, MCP servers, external CLIs)

**What goes here:** Complex workflows that are too long for CLAUDE.md but need to be repeatable and consistent.

### The Tension Between Layers

Their README articulates this explicitly:
> "Hooks are not a security boundary -- a prompt injection can work around them. They are structured prompt injection at opportune times: intercepting tool calls, injecting context, blocking known-bad patterns, and steering agent behavior. Guardrails, not walls."

They know hooks can be bypassed. They use them anyway because they catch the common case. The permission deny list is harder to bypass (it's enforced by the harness, not the model). The CLAUDE.md is the softest layer -- it relies on the model following instructions, which it usually does but not always.

---

## Non-Obvious Insights

### 1. Error Messages as Steering Mechanisms

Every blocking hook includes an actionable alternative in its error message. This is not just UX -- it is prompt engineering. When Claude sees "BLOCKED: Use trash instead of rm -rf", it learns the correct tool and uses it on the next attempt. The hook is a one-shot teaching mechanism.

### 2. Anti-Rationalization Hook (Stop Event)

The README documents a `Stop` hook pattern that uses a prompt-based evaluator (Haiku) to check if Claude is declaring victory while leaving work undone. It catches patterns like:
- "These issues were pre-existing"
- "Fixing this is out of scope"
- "I'll leave these for a follow-up"
- "Too many issues to fix"

This addresses a known failure mode: Claude rationalizing incomplete work. The hook fires after every response, sending it to a fast model for judgment. If rejected, the reason is fed back as Claude's next instruction.

### 3. Crypto Wallet Protection

The deny list includes five specific crypto wallet applications under `~/Library/Application Support/`. This is not paranoia -- it is a concrete threat model. An attacker who can prompt-inject Claude Code (via a malicious repo, MCP server, or package) could exfiltrate wallet data. Trail of Bits defends against this because they audit systems where this attack is realistic.

### 4. Anti-LLM-Speak as a Design Principle

The explicit ban on inflated language in PR descriptions ("avoid: critical, crucial, essential, significant, comprehensive, robust, elegant") is a meta-instruction that improves all downstream output. LLMs default to hyperbolic language. This single instruction produces more honest, readable output across every interaction.

### 5. Agent-Native by Default

Philosophy point #10 -- "Design so agents can achieve any outcome users can" -- is forward-looking. They are building systems where AI agents are first-class citizens alongside human developers. Commands are designed to be runnable by `claude -p` in headless mode. This is not afterthought accessibility; it is a design constraint applied from the start.

### 6. CI as Source of Truth

The commands do not hardcode quality gates. They read CI workflows first and only fall back to defaults when CI is absent. This means the agent enforces the same standards as the project's CI pipeline, not a potentially-stale list in a prompt.

### 7. The Self-Installing Command Pattern

The `/trailofbits:config` command fetches files from GitHub, installs them, and then installs itself. After first use, it works from any directory. This is an elegant distribution mechanism: users clone the repo once, run the command, and never need the repo again. Updates are pulled via the same command.

### 8. Turn Budget Management

The merge-dependabot command explicitly handles running out of turns:
- 75% used: stop new evaluations, merge what you have
- 90% used: print summary and stop

This is resource-aware agent design. Most agent workflows assume infinite context/turns. Trail of Bits designs for the constraint.

### 9. Parallel Multi-Model Review

The review-pr command runs Claude toolkit agents, OpenAI Codex, and Google Gemini in parallel. Different models have different blind spots. The deduplication step merges findings and notes consensus. This is expensive but thorough -- appropriate for a security firm.

### 10. Version-Pinned Regex Patterns

The GAM audit hook comments note that verb lists were "verified against GamCommands.txt v7.33.00." They version their pattern matching against the tool's known command set. When the tool updates, the patterns can be verified against the new command list.

---

## What We Can Learn

### For CLAUDE.md Authoring

1. **Use measurable constraints, not vibes.** "<=100 lines/function" beats "keep functions small."
2. **Negative framing counters LLM additive bias.** Most rules should say what NOT to do.
3. **Bold key + dash explanation** enables scanning. The bold is the rule name; the dash text operationalizes it.
4. **One sentence per idea.** No compound sentences. No filler.
5. **Ban inflated language explicitly.** List the specific words to avoid.
6. **Tables for toolchains.** Tool | replaces | usage -- scannable, unambiguous.
7. **Weave supply chain security into every language section.** Don't make it a separate concern.

### For Hooks

1. **Error messages should instruct, not just block.** "BLOCKED: Use X instead" teaches the correct behavior.
2. **Exit code 2 feeds back to the model.** Use this to redirect, not just prevent.
3. **Handle edge cases in patterns.** Both flag orderings, chained commands, case variants.
4. **Keep hooks focused.** One hook, one concern. The package manager hook is 12 lines.
5. **Hooks are guardrails, not walls.** Layer them with deny lists and CLAUDE.md for defense in depth.

### For Commands

1. **Commands encode organizational processes.** They are the "how we do X" for common workflows.
2. **Design for headless execution.** "Do not stop or ask for confirmation at any step."
3. **Read CI first, fall back to defaults.** The agent should enforce what the project enforces.
4. **Handle resource constraints.** Turn budgets, context limits, tool availability (skip gracefully).
5. **Multi-agent review catches more.** Different models find different issues.

### For Overall Architecture

1. **Three layers: deterministic (settings) > advisory (CLAUDE.md) > procedural (commands).**
2. **Put "must never happen" in settings.json deny lists.** These survive prompt injection.
3. **Put "should usually happen" in CLAUDE.md.** Accept that context can override these.
4. **Put "how to do X" in commands.** Complex workflows that need consistency.
5. **Explicit defaults prevent accidents.** Set `enableAllProjectMcpServers: false` even when it's the default.
6. **Privacy by default.** Disable telemetry unless there's a reason not to.
7. **Think about exfiltration vectors.** What sensitive data exists on this machine? Deny read access to all of it.
