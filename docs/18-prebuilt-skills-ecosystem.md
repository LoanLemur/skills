# Pre-Built Agentic Coding Skills Ecosystem

Research date: 2026-04-03

## Executive Summary

The agentic skills ecosystem has consolidated around a standard format (SKILL.md with YAML frontmatter) and a package manager (`npx skills` from Vercel Labs). Major infrastructure vendors (Vercel, Cloudflare, Netlify, Stripe, HashiCorp) publish official skills. The space has significant noise — dozens of "awesome" lists, hundreds of community skills of variable quality — but a clear tier list emerges.

---

## Tier 1: Official / High-Credibility Sources

### 1. anthropics/skills — Anthropic's Official Skills Repo
- **URL:** https://github.com/anthropics/skills
- **Stars:** 110K | **Forks:** 12.4K | **Commits:** 25
- **License:** Apache 2.0 (example skills), source-available (document skills)

**17 skills included:**
algorithmic-art, brand-guidelines, canvas-design, claude-api, doc-coauthoring, docx, frontend-design (222K installs via skills.sh), internal-comms, mcp-builder, pdf, pptx, skill-creator, slack-gif-creator, theme-factory, web-artifacts-builder, webapp-testing, xlsx

**Pattern:** Each skill is a folder containing `SKILL.md` with YAML frontmatter (`name`, `description`) plus optional scripts and references. This is the canonical format that the entire ecosystem has adopted.

**Assessment:** Reference implementation. The document skills (docx/pdf/pptx/xlsx) are genuinely useful. The creative skills are demos. The `skill-creator` meta-skill is useful for authoring new skills. Repo is actively accepting community PRs.

### 2. Vercel — Three Official Repos + Plugin

**vercel-labs/skills** (The Skills CLI)
- **URL:** https://github.com/vercel-labs/skills
- **Stars:** 12.9K | **Commits:** 245
- **What it is:** `npx skills add <package>` — the npm-style package manager for agent skills. Supports 44+ agents (Claude Code, Cursor, Codex, Copilot, etc.). Feeds from the skills.sh directory.

**vercel-labs/agent-skills** (Vercel's own skills)
- **URL:** https://github.com/vercel-labs/agent-skills
- **Stars:** 24.4K | **Commits:** 191
- **Skills:** react-best-practices (263K installs), web-design-guidelines (213K installs), react-native-guidelines, react-view-transitions, composition-patterns, vercel-deploy-claimable

**Vercel Plugin for Claude Code** (what's running in this session)
- 47+ skills covering Next.js, AI SDK, Turborepo, Functions, Routing Middleware, Storage, Marketplace, etc.
- Delivered as a Claude Code plugin with a relational knowledge graph
- Ships skills for: nextjs, ai-sdk, shadcn, turbopack, vercel-functions, routing-middleware, vercel-storage, vercel-cli, deployments-cicd, env-vars, auth, workflow, runtime-cache, ai-gateway, marketplace, and many more

**Assessment:** Vercel is the most invested vendor in this ecosystem. The `skills` CLI is becoming the de facto distribution mechanism. The plugin (already installed in this project) is the gold standard for vendor-specific skill delivery.

### 3. skills.sh — The Skills Directory/Registry
- **URL:** https://skills.sh
- **Total installs tracked:** 91,478 skills (all-time)
- **Top by installs:** find-skills (787K), vercel-react-best-practices (264K), frontend-design (222K), web-design-guidelines (213K), remotion-best-practices (190K)
- **Top publishers:** Microsoft (2.2M+ installs for Azure/Copilot skills), Vercel Labs, Anthropic

**Assessment:** This is the npm registry equivalent for skills. Useful for discovery. Quality varies wildly — the top publishers (Microsoft, Vercel, Anthropic) are solid; long tail is unvetted.

### 4. Trail of Bits — claude-code-config
- **URL:** https://github.com/trailofbits/claude-code-config
- **Stars:** 1.8K
- **Built by:** Trail of Bits (premier security auditing firm)

**Contents:**
- `settings.json` — Global Claude Code configuration template
- `claude-md-template.md` — System instructions
- `mcp-template.json` — MCP server definitions
- `.claude/commands/trailofbits/` — Custom slash commands
- `hooks/` — Lifecycle hooks (PreToolUse, PostToolUse, Stop)
- `scripts/statusline.sh` — Terminal monitoring

**Security patterns enforced:**
- Blocks reads/writes to SSH keys, cloud credentials (AWS, Azure, K8s), package registry tokens, git credentials, shell configs, macOS Keychain, cryptocurrency wallets
- PreToolUse hooks block `rm -rf` (suggests `trash`) and direct pushes to main/master
- Sandboxing options: OS-level sandbox, devcontainer isolation, remote droplet execution
- Anti-rationalization gates for incomplete work
- Mutation logging and audit trails

**Assessment:** Serious engineering from a credible security firm. The permission deny lists and hook-based guardrails are production-quality patterns worth studying. The sandboxing approach (Seatbelt on macOS, bubblewrap on Linux) is the most rigorous in the ecosystem. Not skills per se — it's a hardened configuration harness.

---

## Tier 2: Vendor-Published Skills (via VoltAgent Registry)

### VoltAgent/awesome-agent-skills
- **URL:** https://github.com/VoltAgent/awesome-agent-skills
- **Stars:** 14K | **Skills:** 1,060+
- **Curation policy:** "Real-world Agent Skills created and used by actual engineering teams, not mass AI-generated stuff"

This registry catalogs official skills from major vendors. The valuable entries:

| Vendor | # Skills | Notable Skills |
|--------|----------|---------------|
| **Stripe** | 2 | stripe-best-practices, upgrade-stripe |
| **Cloudflare** | 6 | agents-sdk, durable-objects, wrangler, web-perf, building-ai-agent-on-cloudflare, building-mcp-server-on-cloudflare |
| **Netlify** | 11-12 | netlify-functions, netlify-edge-functions, netlify-blobs, netlify-db, netlify-image-cdn, netlify-forms, netlify-frameworks, netlify-caching, netlify-config, netlify-cli-and-deploy, netlify-ai-gateway |
| **HashiCorp** | 11 | terraform-style-guide, terraform-stacks, terraform-test, new-terraform-provider, provider-resources, provider-test-patterns, refactor-module, terraform-search-import, azure-verified-modules, provider-actions, run-acceptance-tests |
| **Google Labs (Stitch)** | 6 | design-md, enhance-prompt, + 4 others |
| **Google Workspace** | 8 | CLI tools for Google apps |
| **Vercel** | 7 | (duplicates their own repo — react-best-practices, next-best-practices, next-cache-components, next-upgrade, etc.) |
| **Sentry** | listed | Error tracking integration |
| **Figma** | listed | Design tool integration |
| **Microsoft** | listed | Azure/Copilot skills |
| **Supabase** | listed | Database integration |
| **Neon** | 3 | Postgres-specific skills |

**Assessment:** The registry itself is well-curated. The vendor-published skills within it are the real value — these are maintained by the teams who own the products. HashiCorp's 11 Terraform skills and Netlify's 12 skills are notably comprehensive. Cloudflare's skills cover their newer AI agent platform.

---

## Tier 3: Rails-Specific Skills

### palkan/skills — Layered Rails (Vladimir Dementyev)
- **URL:** https://github.com/palkan/skills
- **Stars:** 284 | **License:** MIT | **Last updated:** Feb 2026
- **Author:** Vladimir Dementyev (Evil Martians, author of "Layered Design for Ruby on Rails Applications", AnyCable creator)

**One skill, six commands:**
- `/layers:analyze` — Full codebase architecture analysis
- `/layers:analyze:callbacks` — Callback scoring and extraction recommendations
- `/layers:analyze:gods` — God object detection
- `/layers:review` — Code change review for layer violations
- `/layers:spec-test` — Specification test application
- `/layers:gradual [goal]` — Incremental adoption planning

**Assessment:** High credibility — palkan is one of the most respected voices in Rails architecture. The skill enforces the patterns from his book. Useful as an architectural linter. Single-purpose and well-executed. The most legitimate Rails skill available.

### ThibautBaissac/rails_ai_agents
- **URL:** https://github.com/ThibautBaissac/rails_ai_agents
- **Stars:** 473 | **Commits:** 61 | **License:** MIT

**Contents:** 18 agents, 23 commands, 13 skills, 12 path-scoped rules, 6 lifecycle hooks

**Agents:** Model, Controller, Service, Migration, Policy, Form, Query, Presenter, ViewComponent, Job, Mailer, Turbo, Stimulus, Tailwind, RSpec, Implementation, TDD Refactoring, Lint

**Skills include:** code-review, security-audit, rails-architecture, authentication-flow, caching-strategies, performance-optimization, action-cable-patterns, active-storage-setup, api-versioning, i18n-patterns, solid-queue-setup, rails-concern, extraction-timing

**Commands include:** Spec-Driven Development pipeline (specify, clarify, spec-review, checklist, plan, tasks, analyze, implement, validate)

**Notable features:**
- Smart model routing (Opus for architecture/security, Sonnet for coding, Haiku for linting)
- Path-scoped rules that auto-load by directory context
- Post-implementation drift detection between code and spec

**Assessment:** Ambitious and comprehensive. The SDD pipeline is an opinionated methodology. The 18-agent approach is architecturally over-engineered for most projects (one agent per Rails concept is excessive), but the individual skills (caching-strategies, solid-queue-setup, etc.) are useful standalone. Worth cherry-picking from, not adopting wholesale.

### obie/claude-on-rails (Obie Fernandez)
- **URL:** https://github.com/obie/claude-on-rails
- **Stars:** 786 | **Releases:** 6 (latest v0.2.0, July 2025) | **Commits:** 37
- **Author:** Obie Fernandez (author of "The Rails Way", Hashicorp VP Engineering alumni)

**Architecture:** Uses claude-swarm to orchestrate 7 specialized agents: Architect, Models, Controllers, Views, Services, Tests, DevOps. Each agent has dedicated prompts and can delegate to others. Integrates Rails MCP Server for live documentation access.

**Assessment:** High-credibility author. The swarm-based architecture is interesting conceptually but depends on claude-swarm (a separate orchestration framework). Relatively young (37 commits). More of an opinionated development framework than a skill collection. Last release is nearly a year old — may not be actively maintained.

### lucianghinda/superpowers-ruby
- **URL:** https://github.com/lucianghinda/superpowers-ruby
- **Stars:** 235 | **Commits:** 417

Ruby/Rails fork of the superpowers project. Includes Ruby 3.x+ idioms, Rails testing with Minitest, Brakeman security scanning, Sandi Metz design rules, 37signals/Basecamp patterns, Hotwire skills, conventional commits.

**Assessment:** Solid Ruby-specific configuration. The Minitest focus and 37signals patterns suggest an opinionated Rails developer. The 417 commits indicate sustained work. More of a CLAUDE.md configuration than discrete skills.

### Shoebtamboli/rails_claude_skills
- **URL:** https://github.com/Shoebtamboli/rails_claude_skills
- A Rails generator gem that scaffolds skill/agent structures into any Rails project. Utility for creating skills, not skills themselves.

---

## Tier 4: Harnesses and Configuration Collections

### Dwarves Foundation — claude-guardrails
- **URL:** https://github.com/dwarvesf/claude-guardrails
- **Stars:** 10 | **Latest release:** v0.3.0 (March 2026)

**Two variants:**
- **Lite:** 3 PreToolUse hooks, 15 deny rules (blocks SSH keys, AWS credentials, .env files, .pem certs)
- **Full:** 5 PreToolUse hooks + PostToolUse scanner, 28 deny rules. Adds GnuPG keys, secrets directories, shell profiles. Includes prompt injection scanner examining command outputs for "ignore previous instructions" patterns.

**Behavioral blocks:** rm -rf, direct pushes, pipe-to-shell, data exfiltration patterns, permission escalation

**Assessment:** Lightweight but functional. The prompt injection scanner is a unique feature not seen in Trail of Bits' config. Very low adoption (10 stars) but the patterns are sound. The lite/full split is practical.

### hesreallyhim/awesome-claude-code
- **URL:** https://github.com/hesreallyhim/awesome-claude-code
- **Stars:** 36.1K | **Forks:** 2.8K | **Commits:** 954

The most popular curated list. Organizes into: Agent Skills, Workflows & Knowledge Guides, Tooling, Status Lines, Hooks, Slash-Commands, CLAUDE.md Files, Alternative Clients, Official Documentation.

**Assessment:** Good discovery resource. Selectively curated (claims editorial standards). The sheer star count reflects community demand for this kind of index. Use it to find things, not as a skills source itself.

### quemsah/awesome-claude-plugins
- **URL:** https://github.com/quemsah/awesome-claude-plugins
- **What it does:** Automated adoption metrics collection using n8n workflows. Indexes 10,428 repositories as of April 2026.
- **Top by subscribers:** prompts.chat (1,609), next.js (1,503), skills/Anthropic (756), claude-code/Anthropic (671)

**Assessment:** Useful as a market intelligence tool. The automated metrics tracking gives objective adoption data vs. curated lists' subjective picks.

---

## What's Actually Worth Using

**For any project:**
1. **Trail of Bits claude-code-config** — Security guardrails. Adapt their deny lists and hooks.
2. **Anthropic's skill-creator skill** — For authoring your own skills in the canonical format.
3. **Vercel's `npx skills` CLI** — For installing vendor skills when needed.

**For Rails projects:**
1. **palkan/skills (Layered Rails)** — Architectural review from the authority on Rails layering.
2. **Cherry-pick from rails_ai_agents** — The individual skills (caching-strategies, solid-queue-setup, action-cable-patterns) are more useful than the full 18-agent framework.

**For specific vendor work:**
- Stripe: stripe-best-practices, upgrade-stripe (via VoltAgent registry or `npx skills`)
- Cloudflare: wrangler, agents-sdk, durable-objects
- Netlify: The full 12-skill suite covers their entire platform
- HashiCorp: 11 Terraform skills are the most comprehensive vendor set
- Vercel: Already installed as the plugin in this project (47+ skills)

**What to skip:**
- Mass-generated skill collections (alirezarezvani/claude-skills with 220+ skills — quantity over quality)
- "Awesome" lists beyond hesreallyhim's (the others are derivative)
- Swarm/multi-agent frameworks (claude-on-rails, claude-007-agents) — orchestration complexity for unclear benefit
- The skills.sh long tail — 91K total installs but the top 5 account for most of the value

---

## Ecosystem Architecture Summary

The stack has settled into three layers:

```
Distribution:  npx skills (Vercel Labs) + skills.sh registry
Format:        SKILL.md with YAML frontmatter (Anthropic standard)
Guardrails:    settings.json hooks + deny rules (Trail of Bits pattern)
```

Skills are SKILL.md files. They install into `.claude/skills/` or equivalent. The format is agent-agnostic — the same skill works across Claude Code, Cursor, Codex, and 40+ other agents. Vendor skills are the most reliable because the vendor maintains them alongside their product.
