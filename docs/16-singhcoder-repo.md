# SinghCoder/claude-code: The "Research Snapshot" Fork

## What It Is

`SinghCoder/claude-code` is a **fork of `ultraworkers/claw-code`** (the 159K-star Rust rewrite) created on March 31, 2026 at 11:29 UTC -- approximately 2.5 hours after the claw-code repo itself appeared. Despite being a fork of claw-code, the repo was immediately repurposed: all 6 commits on the `main` branch are authored by `instructkr` (Sigrid Jin) on the same day, converting the repo into a Python porting workspace.

**Repo description:** "Claude Code Snapshot for Research. All original source code is the property of Anthropic."

- 45 stars, 38 forks as of April 3, 2026
- Homepage links to a tweet by @Fried_rice about the leak
- No license declared
- No issues enabled
- Last pushed: March 31, 2026 (no activity since day one)

## Who Is Behind It

The author is **Sigrid Jin** (`instructkr` / `@Fried_rice`), a Seoul-based engineer profiled by the Wall Street Journal on March 21, 2026 as having consumed 25 billion Claude Code tokens in 2025. The README is heavy on personal narrative -- WSJ quotes, conference attendance, girlfriend's reaction to the leak. Jin frames the work as "harness engineering research."

Jin claims the Python rewrite was done overnight using **oh-my-codex (OmX)**, a workflow layer built on OpenAI Codex by `@bellman_ych` (Yeachan Heo). The README credits OmX's `$team` mode (parallel code review) and `$ralph` mode (persistent execution loops with architect verification).

## Relationship to the March 31 Source Leak

The fork chain is: **Anthropic's npm source map leak -> community mirrors -> `ultraworkers/claw-code` (Rust rewrite, 159K stars) -> `SinghCoder/claude-code` (this fork)**. The repo initially contained the exposed TypeScript snapshot but was converted to a Python workspace on the same day. The README explicitly states: "the exposed snapshot is no longer part of the tracked repository state."

The repo includes an essay dated March 9, 2026 -- *"Is legal the same as legitimate: AI reimplementation and the erosion of copyleft"* -- predating the leak by three weeks, which Jin uses to frame the ethical context of clean-room reimplementation.

## What the Community Could Learn from It

The `src/` directory reveals more than the README admits. While the README describes 8 Python files, the actual `src/` tree contains **50+ files and 25+ subdirectories** mirroring the original Claude Code structure:

- Directories: `assistant/`, `bootstrap/`, `bridge/`, `buddy/`, `cli/`, `components/`, `constants/`, `coordinator/`, `entrypoints/`, `hooks/`, `keybindings/`, `memdir/`, `migrations/`, `native_ts/`, `outputStyles/`, `plugins/`, `reference_data/`, `remote/`, `schemas/`, `screens/`, `server/`, `services/`, `skills/`, `state/`, `types/`, `upstreamproxy/`, `utils/`, `vim/`, `voice/`
- Notable files: `parity_audit.py` (5.2KB -- compares Python port against the original), `runtime.py`, `context.py`, `cost_tracker.py`, `history.py`

This directory listing is arguably the most useful artifact: it provides a **structural map of Claude Code's internal module organization** (coordinator, buddy, bridge, memdir, hooks, skills, voice, vim, etc.) even without the implementation code. The `parity_audit.py` file explicitly compares the Python workspace against an "ignored archive" of the original source.

## How It Differs from claw-code

| Dimension | claw-code (ultraworkers) | SinghCoder/claude-code |
|---|---|---|
| Language | Rust | Python |
| Scale | 159K stars, massive community | 45 stars, niche |
| Approach | Full runtime-equivalent rewrite | Structural scaffold / porting workspace |
| Completeness | Actively maintained, near-parity | Metadata and stubs, not executable |
| Tooling | Community-driven | AI-assisted via OmX (OpenAI Codex wrapper) |
| Status | Active development, ownership transfer in progress | Abandoned after day one |
| Parent | Original mirrors of the leaked source | Fork of claw-code itself |

The key difference: claw-code aims to be a working replacement. SinghCoder's repo is a **porting scaffold** -- it catalogs what needs to be ported (commands, tools, subsystems) and provides metadata about progress, but contains minimal executable logic. The `query_engine.py` and `port_manifest.py` are introspection tools for tracking porting status, not functional agent code.

## Current State

**Effectively abandoned.** All 6 commits occurred on March 31, 2026. No commits since. No issues. The `instructkr/claude-code` repo referenced in the README does not exist (returns 404). The repo remains a frozen snapshot of a single day's work -- an ambitious overnight sprint that produced scaffolding but no functional agent runtime.

The 38 forks suggest some community interest, but none of the forks have significant star counts. The repo's lasting value is as a structural reference map of Claude Code's module organization, not as a working codebase.
