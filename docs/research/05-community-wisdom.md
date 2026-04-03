# Community Wisdom: What Experienced Engineers Say About Agentic Coding

*Research compiled April 2026. Sources span late 2024 through April 2026.*

---

## 1. The Authoritative Voices

### Anthropic's Official Best Practices

Anthropic's own documentation ([code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)) is the single most comprehensive source. The core philosophy:

> "Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."

Key official recommendations:
- **Verification is the #1 lever.** "Claude performs dramatically better when it can verify its own work -- run tests, compare screenshots, validate outputs." Provide tests, linter configs, or bash commands that check output.
- **Explore -> Plan -> Implement -> Commit.** Use Plan Mode to separate research from execution. Skip planning only when the change is trivially scoped.
- **CLAUDE.md is code.** Keep it concise. For each line, ask: "Would removing this cause Claude to make mistakes?" If not, cut it. Bloated CLAUDE.md causes Claude to ignore instructions. Check it into git. Prune regularly.
- **Context is the fundamental resource.** Use `/clear` between unrelated tasks. Delegate research to subagents (separate context windows). After two failed corrections, `/clear` and start fresh with a better prompt.
- **Common failure patterns:** The kitchen sink session (mixing unrelated tasks), correcting over and over (polluted context), the over-specified CLAUDE.md, the trust-then-verify gap, and infinite exploration.

Source: [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)

### Boris Cherny (Staff Engineer at Anthropic, creator of Claude Code)

Cherny's CLAUDE.md is ~100 lines. His three core principles:
1. **Simplicity first** -- minimize code changes; delete rather than add when possible
2. **No laziness** -- address root causes; avoid temporary fixes
3. **Minimal impact** -- only modify what's necessary

He runs 10-15 Claude sessions simultaneously, each with dedicated git worktrees. His golden rule: "Anytime we see Claude do something incorrectly, we add it to CLAUDE.md so it doesn't repeat next time." The file is checked into git, shared by the whole team, and updated multiple times a week.

Prompting style: describes desired outcomes, doesn't micromanage. "Prove to me this works" or simply "Fix" with a bug report.

Source: [Claude Code Best Practices: Inside the Creator's 100-Line Workflow](https://mindwiredai.com/2026/03/25/claude-code-creator-workflow-claudemd/)

### Anthropic's 2026 Agentic Coding Trends Report

Key data points:
- 78% of Claude Code sessions in Q1 2026 involve multi-file edits (up from 34% in Q1 2025)
- Average session length: 23 minutes (up from 4 minutes)
- Developers use AI in ~60% of their work, but can fully delegate only 0-20% of tasks
- Engineers tend to delegate tasks that are "easily verifiable" or low-stakes

The report identifies a shift: "Software development is shifting from an activity centered on writing code to an activity grounded in orchestrating agents that write code."

Sources: [2026 Agentic Coding Trends Report](https://resources.anthropic.com/2026-agentic-coding-trends-report) | [Eight trends blog post](https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026)

---

## 2. Independent Practitioners

### Simon Willison -- Agentic Engineering Patterns

Willison's [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/) guide is the most thoughtful independent resource. Published as a series of pattern chapters (inspired by the 1994 Design Patterns book), it covers:

**Core philosophy:** Agentic engineering describes "professional software engineers using coding agents to improve and accelerate their work by amplifying their existing expertise." It is explicitly not vibe coding.

**Red/green TDD:** His most actionable pattern. Tell the agent: "Build a Python function to extract headers from a markdown string. Use red/green TDD." Every good model understands "red/green TDD" as shorthand. The agent writes tests, confirms they fail (red), then implements until they pass (green). This prevents two major risks: code that doesn't work, and unnecessary code.

**AI should help produce better code:** Shipping worse code with agents is "a choice." Use agents for refactoring -- the simple-but-time-consuming work like API redesigns, naming corrections, removing duplication. Run agents asynchronously in branches. Evaluate results through PRs: "If it's good, land it. If it's almost there, prompt it. If it's bad, throw it away."

**Git with coding agents:** Don't memorize Git mechanics. Stay aware of Git's capabilities and delegate the operational details. Agents excel at resolving merge conflicts, running `git bisect`, rewriting history, and extracting libraries while preserving commit history.

**On the podcast with Lenny Rachitsky (April 2026):** "The only universal skill is being able to roll with the changes." The bottleneck has shifted from coding to testing. Build three UI prototypes instead of one since "a UI prototype is free now." Using these tools effectively "takes a lot of practice."

Sources: [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/) | [Red/green TDD](https://simonwillison.net/guides/agentic-engineering-patterns/red-green-tdd/) | [Better code](https://simonwillison.net/guides/agentic-engineering-patterns/better-code/) | [Git with agents](https://simonwillison.net/guides/agentic-engineering-patterns/using-git-with-coding-agents/) | [Lenny's Podcast](https://simonwillison.net/2026/Apr/2/lennys-podcast/)

### Kent Beck -- Augmented Coding

Beck distinguishes **augmented coding** from vibe coding. In augmented coding, "you care about the code, its complexity, the tests, & their coverage." In vibe coding, you don't.

His framework for AI-assisted TDD:
- System prompt mandates strict TDD: "Always follow the TDD cycle: Red -> Green -> Refactor"
- Implement only the minimum code to pass each test
- **Never mix structural and behavioral changes in the same commit**
- Watch for three warning signs: loops in implementation, unrequested functionality, test manipulation or deletion ("AI agents keep trying to delete tests to make them pass")

He constrain context by "only telling the AI what it needs for the next step," preserves optionality, and balances expansion (features) with contraction (refactoring). He built a production-competitive B+ Tree library in Rust and Python using this method.

His broader insight: developers "make more consequential decisions per hour while handling fewer routine tasks." The economics have changed -- previously expensive activities are now cheap, but discipline is more important, not less.

Sources: [Augmented Coding: Beyond the Vibes](https://tidyfirst.substack.com/p/augmented-coding-beyond-the-vibes) | [TDD, AI agents and coding with Kent Beck (Pragmatic Engineer)](https://newsletter.pragmaticengineer.com/p/tdd-ai-agents-and-coding-with-kent) | [O11ycast Ep. 80](https://www.heavybit.com/library/podcasts/o11ycast/ep-80-augmented-coding-with-kent-beck)

### Adam Tornhill (Founder/CTO of CodeScene) -- Code Health as AI Readiness

Tornhill's team reports 2-3x speedup on tasks after adopting agentic AI, but warns:

> "Speed amplifies both good design and bad decisions, which means code health, automated safeguards, and short feedback loops become the real enablers of progress."

Key data point: AI requires a Code Health score of at least 9.5 (ideally 10.0) for reliable operation. Poor code health increases both defect risk and token waste. Uplifted codebases see ~50% lower token consumption.

His six operational patterns:
1. **Risk assessment** before deploying agents to a codebase
2. **Continuous safeguards** -- pre-commit and PR-level code health checks
3. **Refactoring** to expand AI-ready codebase surface area
4. **Encoding rules** -- document workflows in AGENTS.md
5. **Coverage gates** -- code coverage as behavioral regression signals
6. **End-to-end testing** -- automate full system validation

Source: [Agentic AI Coding: Best Practice Patterns for Speed with Quality](https://codescene.com/blog/agentic-ai-coding-best-practice-patterns-for-speed-with-quality)

### Harper Reed -- Spec-First Workflow

Reed's workflow (originally Feb 2025, later migrated to Claude Code) follows three phases:

1. **Idea honing:** Use a conversational LLM with the prompt "Ask me one question at a time" to develop a thorough spec. Save as `spec.md`.
2. **Planning:** Use a reasoning model to produce `prompt_plan.md` (step-by-step prompts) and `todo.md` (trackable checklist).
3. **Execution:** Implement prompts iteratively. "Aggressively keep track of what's going on because you can easily get ahead of yourself."

He takes charge of initial boilerplate and tooling setup, since "Claude has a tendency to output react code and having a solid foundation helps."

Sources: [My LLM codegen workflow atm](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/) | [Basic Claude Code](https://harper.blog/2025/05/08/basic-claude-code/)

### Ran Isenberg (Principal Software Architect, Palo Alto Networks)

After months of daily Claude Code usage: "Your domain expertise is the bottleneck, not the tool."

Practical advice:
- Keep CLAUDE.md under 200 lines
- Use Opus for complex tasks, Sonnet for planning/iteration
- Invest in specs before code -- always worth the time
- Prefer community skills over MCP servers (skills are transparent; MCPs are "black boxes" you can't review for security)
- For substantial projects, use a structured methodology (e.g., BMAD) before Claude Code

Source: [Claude Code Best Practices: Lessons From Real Projects](https://ranthebuilder.cloud/blog/claude-code-best-practices-lessons-from-real-projects/)

### incident.io Team -- Git Worktrees in Practice

After four months from research preview to production use:
- Adopted git worktrees for parallel Claude Code sessions
- Used voice dictation (SuperWhisper) for complex feature specs
- A JavaScript editor enhancement took 10 minutes vs. estimated 2 hours
- Built custom bash functions to reduce friction

Key insight: "The real value we've found isn't in how to craft the perfect prompt, but in sharing learnings and in the workflows and tools we've built around Claude."

Source: [Shipping faster with Claude Code and Git Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees)

---

## 3. The Contrarian and Cautionary Voices

### The METR Study: AI Actually Slows Down Experienced Developers

The most rigorous study to date. METR ran a randomized controlled trial with 16 experienced open-source developers on 246 real issues.

**Finding: Developers were 19% slower with AI tools, despite believing they were 20% faster.**

- Developers accepted less than 44% of AI generations
- AI introduced "extra cognitive load and context-switching"
- The repositories averaged 10 years old with 1M+ lines of code
- The perception gap is the most striking part: subjective experience of speed doesn't match objective measurement

METR is redesigning their experiment as of Feb 2026, acknowledging limitations. But this remains the strongest evidence that AI tools require deliberate practice to be net-positive for experienced developers on familiar codebases.

Sources: [METR study](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) | [Sean Goedecke's analysis](https://www.seangoedecke.com/impact-of-ai-study/) | [arxiv paper](https://arxiv.org/abs/2507.09089)

### Margaret-Anne Storey -- Cognitive Debt

Storey (professor, coined the term at a Thoughtworks retreat) argues the primary risk shifts from technical debt to cognitive debt: "the loss of shared understanding among developers about what software does and why design decisions were made."

Drawing on Peter Naur's "Programming as Theory Building": a program is a theory that lives in developers' minds. AI-generated code fragments this understanding. She illustrates with a student team that shipped features quickly but "no one could explain why certain design decisions had been made," leading to inability to make even simple changes.

Warning signs: team members hesitating to modify code, over-reliance on tribal knowledge, treating systems as black boxes.

Recommendations:
- Require at least one human fully understands each AI-generated change
- Document reasoning, not just what changed
- Conduct regular sessions to rebuild shared understanding
- Accept that "velocity without understanding is not sustainable"

Sources: [Cognitive Debt blog post](https://margaretstorey.com/blog/2026/02/09/cognitive-debt/) | [Simon Willison's coverage](https://simonwillison.net/2026/Feb/15/cognitive-debt/)

### Addy Osmani -- Comprehension Debt and the 80% Problem

Osmani (Google Chrome team) extends Storey's concept. Comprehension debt is "the growing gap between how much code exists in your system and how much of it any human being genuinely understands."

Key data:
- An Anthropic study of 52 engineers: AI-assisted participants scored 17% lower on comprehension quizzes, with the largest declines in debugging
- CodeRabbit 2026 analysis: teams merged 98% more PRs that were 154% larger year-over-year
- 61% of developers report AI produces code that "looks correct but is unreliable"
- "A junior engineer can now generate code faster than a senior engineer can critically audit it"

Critical distinction: "Passive delegation ('just make it work') impairs skill development far more than active, question-driven use of AI."

**The 80% Problem:** Agents can rapidly generate 80% of code, but the remaining 20% -- integration, subtle bugs, performance tuning -- requires deep understanding. Agents "overcomplicate relentlessly, scaffolding 1,000 lines where 100 would suffice." They "don't push back" on incomplete or contradictory instructions.

Recommendations:
- Write tests before requesting implementation
- Fresh-context code review (ask the same model to review with a clean context)
- Spend 70% of effort on problem definition, 30% on execution
- Treat AI-generated code like mentorship material -- investigate what you don't understand
- Maintain system-level mental models, not just line-by-line review

Sources: [Comprehension Debt](https://addyosmani.com/blog/comprehension-debt/) | [The 80% Problem](https://addyo.substack.com/p/the-80-problem-in-agentic-coding)

### Security Research: AI Code Has 2.74x More Vulnerabilities

Veracode tested 100+ AI models across 80 coding tasks: only 55% of AI-generated code was secure. AI-generated code has 2.74x more vulnerabilities than human-written code.

- Java: 72% security failure rate
- Cross-Site Scripting (CWE-80): failed in 86% of cases
- Log Injection (CWE-117): failed in 88% of cases
- By June 2025, AI-generated code was adding 10,000+ new security findings per month (10x increase from Dec 2024)
- Apiiro found 153% increase in design-level security flaws in AI code
- **Newer and larger models don't generate significantly more secure code than their predecessors**

Sources: [Veracode GenAI Code Security Report](https://www.veracode.com/resources/analyst-reports/2025-genai-code-security-report/) | [Apiiro research](https://apiiro.com/blog/4x-velocity-10x-vulnerabilities-ai-coding-assistants-are-shipping-more-risks/)

---

## 4. Hacker News Practitioner Wisdom

From the [Claude Code: Best practices for agentic coding](https://news.ycombinator.com/item?id=43735550) HN discussion:

- **jasonjmcghee:** "Tell it to read specific files (only those!)" to avoid unnecessary token consumption. Disable auto-formatting during sessions (busts cache). Avoid manual file edits mid-session.
- **BeetleB (contrarian):** Prefers Aider because "you explicitly specify the files. You don't have to do _work_ to _limit_ context." Suggests effective agentic coding might require less optimization theater than advertised.
- **pclmulqdq:** Engineers spending under $0.50 per task attribute success to "pretty good code hygiene" and clear instructions -- the tool rewards discipline.
- **bob1029:** Comparing AI coding to outsourcing: you still need expert judgment. Defining policies matters less than "a skilled human developer leverage it for immediate questions."
- **Multiple commenters:** Claude Code is "overeager" -- rewrites working code unnecessarily. The real cost isn't tokens but time spent damage-controlling the agent's choices.

From the [Collation of Claude Code Best Practices v2](https://news.ycombinator.com/item?id=45830267) discussion:
- Keep CLAUDE.md files under 200 lines per file (60 lines when possible)
- Wrap domain-specific rules in tags
- Use multiple CLAUDE.md files for monorepos

---

## 5. Industry Survey Data (Gergely Orosz / Pragmatic Engineer)

From [AI Tooling for Software Engineers in 2026](https://newsletter.pragmaticengineer.com/p/ai-tooling-2026):

- 95% of respondents use AI tools at least weekly
- 75% use AI for at least half their software engineering work
- 70% use 2-4 AI tools simultaneously
- 55% regularly use AI agents
- Staff+ engineers are the heaviest agent users (63.5%)
- Claude Code has overtaken GitHub Copilot as the most-used AI coding tool
- Both Martin Fowler and Kent Beck said "things haven't shifted so rapidly during their 50+ years in the industry"

---

## 6. Emerging Consensus: What The Smart People Agree On

Across all sources, a clear consensus emerges:

### The fundamentals matter more, not less
Every serious practitioner says the same thing: agentic coding requires *more* engineering discipline, not less. Testing, code health, architecture, and specification all become more important when code is cheap to generate.

### Spec before code
Harper Reed, Boris Cherny, Ran Isenberg, Addy Osmani, and Anthropic's own docs all converge on: invest heavily in defining the problem before letting the agent write code. The ratio Osmani recommends: 70% problem definition, 30% execution.

### TDD is the killer pattern
Kent Beck, Simon Willison, and Anthropic's docs all elevate TDD (specifically red/green TDD) as the single most effective practice for agentic coding. Tests give agents a verification loop and prevent regressions.

### Context management is the core skill
Every source treats context window management as the #1 operational concern. `/clear` between tasks, subagents for research, fresh sessions for new work, keeping CLAUDE.md lean.

### Your expertise is the bottleneck
Ran Isenberg says it plainly. The METR study proves it: without deep codebase knowledge, AI tools can actually slow you down. Domain expertise is what makes agent output usable.

### Review everything, trust nothing
The security data is damning (2.74x more vulnerabilities). The comprehension debt research is sobering (17% lower understanding). The METR study shows experienced developers overestimate AI's help. Every serious practitioner reviews all generated code.

### Parallel sessions with git worktrees
Boris Cherny (10-15 sessions), incident.io, and the broader community have converged on git worktrees as the standard pattern for parallel agent work. 3-5 simultaneous worktrees is the practical upper bound before context-switching overhead dominates.

---

## 7. What's Still Contested

- **How much planning is too much?** Harper Reed does extensive spec -> plan -> execute. Others say skip planning for small tasks. No clear threshold.
- **MCP servers vs. skills.** Isenberg prefers skills (transparent, reviewable). Others embrace MCP for external integrations. Security concerns are real but unresolved.
- **Whether AI tools are net-positive for experienced developers on mature codebases.** The METR study says no. Most practitioners say yes. The truth likely depends on how you use them and what "experienced" means in context.
- **How many parallel agents before diminishing returns.** Boris Cherny runs 10-15. Most practitioners say 3-5. The right number depends on task independence and review capacity.
- **Whether agent-written code should be reviewed differently than human code.** Some say yes (watch for specific failure modes like abstraction bloat and assumption propagation). Others say good code review is good code review regardless of author.
