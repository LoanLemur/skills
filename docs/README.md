# Research

How we arrived at these skills. Four rounds of parallel research agents surveyed 150+ sources across the agentic coding landscape (April 2026), followed by verification, source spot-checks, adversarial red teaming, and craft analysis of the best existing skills.

## Start Here

- **[00-synthesis.md](00-synthesis.md)** — The unified spec. Principles, pipeline, skill architecture, craft guide. This is the document that informed every skill in the repo.
- **[28-skill-map.md](28-skill-map.md)** — The final skill map. What we're building, why, and in what order.

## Research Documents

### Round 1: Landscape and Best Practices (docs 01-07)

| Doc | What It Covers |
|---|---|
| [01](01-tool-landscape.md) | 12 tools surveyed: Claude Code, Codex, Cursor, Kiro, Antigravity, Aider, Cline, Devin, Factory, Augment, Copilot, Windsurf |
| [02](02-spec-driven-dev.md) | Spec-driven development: Kiro, Antigravity, GitHub Spec Kit, BMAD, AGENTS.md |
| [03](03-implementation.md) | Implementation patterns: decomposition, multi-agent, git workflows, context management |
| [04](04-quality-assurance.md) | Quality: TDD, code review, verification, guard rails, trust spectrum |
| [05](05-community-wisdom.md) | Practitioner opinions: Willison, Beck, Osmani, Tornhill, Cherny |
| [06](06-anti-patterns.md) | Anti-patterns: vibe coding failures, production incidents, security |
| [07](07-enterprise-vs-startup.md) | Team approaches: FAANG, startup, OSS, consulting, Codex community |

### Round 2: Gap Fills (docs 08-14)

| Doc | What It Covers |
|---|---|
| [08](08-agent-skills-standard.md) | The Agent Skills open standard: spec, adoption, ecosystem |
| [09](09-source-leak-insights.md) | Claude Code source leak: sub-agent models, prompt caching, memory |
| [10](10-anthropic-official-docs.md) | Anthropic official docs: hooks, subagents, plugins, auto mode |
| [11](11-language-framework-impact.md) | Language/framework impact on AI code quality |
| [12](12-missed-practitioners.md) | Missed sources: gold standard file pattern, adversarial testing |
| [13](13-self-improving-skills.md) | Self-improving skills: the maturity spectrum |
| [14](14-ruby-rails-impact.md) | Ruby on Rails and AI coding agents |

### Round 3: Verification and Adversarial Review (docs 15-19)

| Doc | What It Covers |
|---|---|
| [15](15-verification-report.md) | Quality audit: 7 failures found, all fixed in synthesis v2 |
| [16](16-singhcoder-repo.md) | The SinghCoder/claude-code fork |
| [17](17-red-team-report.md) | Adversarial review of every principle: 4 STRONG counterarguments |
| [18](18-prebuilt-skills-ecosystem.md) | Pre-built skills from Anthropic, Vercel, Trail of Bits, vendors |
| [19](19-ralph-loop.md) | Ralph Loop: autonomous iteration with fresh context |

### Round 4: Craft Analysis (docs 20-29)

| Doc | What It Covers |
|---|---|
| [20](20-anthropic-skill-analysis.md) | How Anthropic writes skills |
| [21](21-vercel-skill-analysis.md) | How Vercel writes skills (263K installs) |
| [22](22-trailofbits-analysis.md) | Trail of Bits security harness |
| [23](23-community-skill-analysis.md) | Community skills: Superpowers, palkan, HashiCorp, BMAD |
| [24](24-skill-writing-craft.md) | The meta-skill: 10 principles of writing skills well |
| [25](25-practitioner-skill-analysis.md) | Analysis of existing define/design/execute pipeline |
| [26](26-completion-bias.md) | Agent completion bias: causes, solutions |
| [27](27-practitioner-pain-points.md) | Lived experience: sycophancy, verbosity, rule drift |
| [28](28-skill-map.md) | Final skill map |
| [29](29-agents-vs-skills.md) | When to use agents vs skills, persona effectiveness |
