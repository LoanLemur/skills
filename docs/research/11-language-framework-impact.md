# Does Language/Framework Choice Affect AI Code Quality?

**Summary:** Yes, significantly. Programming language choice creates 5-18x performance gaps in AI agent benchmarks, type systems catch the dominant class of AI errors at compile time, framework opinions dramatically improve code consistency, and codebase health is the single strongest predictor of AI agent success.

---

## 1. Benchmark Data: AI Performance Varies Dramatically by Language

### Multi-SWE-bench (ByteDance, 2025)

The most comprehensive multilingual AI coding benchmark: 2,132 instances across 8 languages, tested with 9 models across 3 agent frameworks.

**Claude 3.7 Sonnet resolution rates by language (MopenHands framework):**

| Language | Resolution Rate |
|----------|----------------|
| Python | 52.2% |
| Java | 21.9% |
| Rust | 15.9% |
| C++ | 14.7% |
| C | 8.6% |
| Go | 7.5% |
| JavaScript | 5.1% |
| TypeScript | 2.2% |

Python-to-TypeScript gap: **24x**. Python-to-Java gap: **2.4x**.

Key finding: "SOTA agents that achieve 52% on Python drop to 4-8% on Go/Rust and <10% on JavaScript/TypeScript/C/C++." The authors attribute this to training data concentration in Python-centric benchmarks.

Source: [Multi-SWE-bench: A Multilingual Benchmark for Issue Resolving](https://arxiv.org/abs/2504.02605)

### SWE-bench Multilingual (Princeton, 2025)

300 tasks across 42 repositories, 9 languages. Claude 3.7 Sonnet + SWE-agent:

| Language | Resolution Rate |
|----------|----------------|
| Rust | 58.1% |
| Java | 53.5% |
| PHP | 48.8% |
| Ruby | 43.2% |
| JS/TS | 34.9% |
| Go | 31.0% |
| C/C++ | 28.6% |

Overall: 42.7% (vs 63% on Python-only SWE-bench Verified).

Notable: Rust performs *best* here, likely because Rust repositories have exceptional test coverage and clear compiler feedback, giving the agent a tight verification loop.

Source: [SWE-bench Multilingual](https://www.swebench.com/multilingual.html)

### SWE-PolyBench (Amazon, 2025)

2,110 instances across Java, JavaScript, TypeScript, Python. Atlassian Rovo Dev (top agent):

| Language | Resolution Rate |
|----------|----------------|
| Python | 54.9% |
| Java | 50.0% |
| TypeScript | 49.0% |
| JavaScript | 43.0% |

Smaller gaps here, but Python still leads. TypeScript outperforms JavaScript consistently.

Source: [SWE-PolyBench](https://amazon-science.github.io/SWE-PolyBench/)

### Key Takeaway

Performance hierarchy varies by benchmark, but two patterns are consistent:
1. **Python dominates** due to training data concentration
2. **Languages with strong type systems or compiler feedback** (Rust, Java, TypeScript) tend to outperform their less-typed counterparts (C, JavaScript)

The Multi-SWE-bench results are the most alarming: models that seem capable on Python-heavy benchmarks may be nearly useless on other languages.

---

## 2. Security: Language Choice Creates Massive Vulnerability Gaps

### Veracode 2025 GenAI Code Security Report

100+ LLMs tested across 80 security-focused coding tasks in Java, JavaScript, Python, and C#.

**Security failure rates by language:**

| Language | Failure Rate |
|----------|-------------|
| Java | >70% |
| Python | 38-45% |
| C# | 38-45% |
| JavaScript | 38-45% |

Java's failure rate is nearly **2x** other languages. The report attributes this to Java's verbose security APIs (cryptography, input validation) that models consistently misconfigure.

**Vulnerability-specific failure rates (all languages):**
- Log injection (CWE-117): 88% failure
- Cross-site scripting (CWE-80): 86% failure
- SQL injection (CWE-89): tested
- Insecure crypto (CWE-327): tested

**Model size does not help:** "Whether you're using a 20-billion or 400-billion parameter model, security performance clusters around the same disappointing 55% mark."

Source: [Veracode 2025 GenAI Code Security Report](https://www.veracode.com/blog/genai-code-security-report/)

### Veracode Spring 2026 Update

GPT-5 achieved 70-72% pass rates (up from 50-60% average), but most models still hover at 50-59%. Java security remains "stubbornly low" while other languages show modest improvement.

Source: [Veracode Spring 2026 Update](https://www.veracode.com/blog/spring-2026-genai-code-security/)

---

## 3. Type Systems: The Strongest Automated Defense Against AI Errors

### The 94% Statistic

Research from ETH Zurich and UC Berkeley found that **94% of compilation errors in LLM-generated code are type-related errors** — the exact class of error that TypeScript catches automatically.

Source: [arXiv:2504.09246](https://arxiv.org/abs/2504.09246), cited in [GitHub Blog](https://github.blog/news-insights/octoverse/typescript-python-and-the-ai-feedback-loop-changing-software-development/)

### TypeScript's Rise as the AI-Era Language

GitHub Octoverse 2025: TypeScript overtook both JavaScript and Python as the #1 language on GitHub. Key data:
- 66% year-over-year growth (2.6M+ monthly contributors)
- "AI models tend to perform better on languages which expose information about correctness, like a type system"
- TypeScript catches 38% of bugs at compile time that JavaScript surfaces only in production

Source: [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)

### How Types Help AI Agents

Types serve as machine-readable specifications. When an AI agent can read type definitions, it:
- Generates more accurate code (types constrain the solution space)
- Gets immediate compiler feedback on mistakes (tight verification loop)
- Produces code that integrates correctly with existing systems (interface contracts)

The practical effect: developers accept larger AI suggestions in TypeScript because the compiler catches integration errors. In JavaScript, each suggestion requires manual verification.

Source: [Builder.io: TypeScript vs JavaScript](https://www.builder.io/blog/typescript-vs-javascript)

### Rust's Compiler as an AI Guardrail

Rust's borrow checker and type system create an unusually strong verification loop. In SWE-bench Multilingual, Rust achieved the highest resolution rate (58.1%) — likely because the compiler gives agents specific, actionable error messages that enable self-correction.

However, AI still struggles with Rust's more complex patterns (lifetimes, trait bounds), and suggestions frequently need adjustment to satisfy the borrow checker.

Source: [Rust forum discussion](https://users.rust-lang.org/t/using-ai-to-generate-rust-code/128758)

---

## 4. Framework Characteristics That Help or Hurt

### Opinionated Frameworks Produce Better AI Code

When frameworks dictate one way to do things (convention over configuration), AI follows the pattern consistently. When frameworks are unopinionated, AI makes different architectural choices on every prompt.

**Framework AI-friendliness ranking (from Encore's analysis):**

| Framework | Convention Strength | AI Consistency |
|-----------|-------------------|----------------|
| Encore | Very high (infrastructure-from-code) | High |
| NestJS | High (decorators, modules) | Medium-High |
| Next.js | High (file-based routing, conventions) | Medium-High |
| Rails/Django/Laravel | High (convention over config) | Medium |
| Fastify | Low (schema validation only) | Low |
| Express | Very low (everything is middleware) | Low |

Express produces the most inconsistent AI results despite having the most training data, because developers must make too many architectural decisions independently.

Source: [Encore: Best Frameworks for AI-Assisted Development](https://encore.dev/articles/best-frameworks-ai-assisted-development)

### But Type Safety Trumps Conventions

Encore's analysis ranks type-safe frameworks above convention-heavy but dynamically-typed ones: "The stricter the types, the more AI mistakes get caught without human intervention." Rails/Django/Laravel rank lower than NestJS/Next.js despite stronger conventions because runtime-only validation lets errors reach production.

### Next.js: Framework-Level AI Agent Support

Next.js has invested heavily in AI agent support, achieving measurable results:

- **AGENTS.md** (bundled docs index): 100% pass rate on Next.js API evals vs 79% for skill-based approaches
- **MCP integration**: Exposes runtime state (errors, routes, segments) directly to agents
- **Agent DevTools** (v16.2): Terminal access to React DevTools diagnostics
- **Docs in node_modules**: Full documentation as plain Markdown, always available to agents

Key insight: "Always-available context works better than on-demand retrieval, because agents often fail to recognize when they should search for documentation."

Sources: [Next.js Agentic Future](https://nextjs.org/blog/agentic-future), [Next.js 16.2](https://nextjs.org/blog/next-16-2-ai), [AGENTS.md outperforms skills](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)

### Rails/Django: Convention Strength, Documentation Debt

Traditional opinionated frameworks (Rails, Django, Laravel) have strong conventions that align well with AI pattern-following, but they lack the framework-level AI agent support that Next.js has built. Their advantage is massive training data from decades of open-source code and documentation.

---

## 5. Codebase Health: The Strongest Predictor of AI Success

### CodeScene's Research

Peer-reviewed research on how codebase quality determines AI agent effectiveness:

- **Code Health threshold for AI:** 9.5 out of 10 (ideally 10.0). Human-readable threshold is 9.0.
- **Industry average:** 5.15 out of 10 — far below the AI-friendly threshold
- **Defect risk in unhealthy code:** AI coding assistants increase defect risk by **at least 30%** when applied to code below 7.0 health score
- **Healthy code benefits:** Up to 15x fewer defects, 50% lower token consumption for equivalent tasks
- **Refactoring effectiveness:** MCP-guided agents achieve 2-5x more Code Health improvements; fix rates reach 90-100% with CodeHealth-aware guardrails
- **Real-world scaling:** loveholidays went from 0 to 50% agent-assisted code in 5 months while maintaining quality, using code health guardrails

Key finding: "AI accelerates output but cannot distinguish 'working' code from 'maintainable' code."

Sources: [CodeScene: Agentic AI Coding Best Practices](https://codescene.com/blog/agentic-ai-coding-best-practice-patterns-for-speed-with-quality), [CodeScene press release](https://www.prnewswire.com/news-releases/ai-coding-assistants-increase-defect-risk-by-30-in-unhealthy-code-new-peer-reviewed-research-finds-302672355.html)

### GitClear's Code Quality Metrics (211M Lines Analyzed)

Tracking AI's impact on code quality across 211 million lines of structured change data (2020-2024):

- **Code churn** (new code revised within 2 weeks): rose from 3.1% (2020) to 5.7% (2024)
- **Code duplication**: increased from 8.3% to 12.3% of changed lines (~4x growth in duplicated blocks)
- **Refactoring**: dropped from 25% to under 10% of changed lines
- **PR density**: median PR size increased 33% (57 to 76 lines) per Greptile's data

Source: [GitClear AI Copilot Code Quality 2025](https://www.gitclear.com/ai_assistant_code_quality_2025_research)

### Qodo: Context Is the Key Quality Driver

From Qodo's 2025 State of AI Code Quality report (survey of developers):

- **88%** of developers have low confidence in shipping AI-generated code
- **25%** estimate 1 in 5 AI suggestions contain factual errors
- **65%** report missing context as the top issue during refactoring
- **81%** quality improvement when AI review is in the loop (vs 55% without)
- AI-generated PRs contain **1.7x more issues** than human-written ones (10.83 vs 6.45 per PR)

Source: [Qodo State of AI Code Quality 2025](https://www.qodo.ai/reports/state-of-ai-code-quality/)

---

## 6. Test Infrastructure Impact

No controlled studies directly compare test infrastructure quality to AI agent performance, but the evidence is strong from adjacent data:

### Verification Loops Are Essential

The SWE-bench Multilingual results show Rust (58.1%) outperforming languages with far more training data. The likely explanation: Rust's compiler provides specific, actionable feedback that enables agent self-correction. This is a verification loop — the agent generates code, the compiler catches errors, and the agent iterates.

Languages/frameworks with strong test infrastructure create similar loops:
- **Compile-time checks** (TypeScript, Rust, Go): fastest feedback, catches 94% of type errors
- **Test suites**: agents can run tests and iterate on failures
- **Linters/formatters**: enforce consistency automatically

### Enterprise Complexity Degrades Performance

Complex codebases degrade AI performance regardless of language:
- Enterprise codebases can be 100B+ lines (4 orders of magnitude larger than Linux)
- Even the best agents resolve only 11-12.5% of complex feature tasks (FeatureBench)
- Performance drops sharply when fixes exceed 600 tokens or touch multiple files
- Cross-file issues are significantly harder for all models

Source: [FeatureBench](https://arxiv.org/html/2602.10975v1), [Multi-SWE-bench](https://arxiv.org/abs/2504.02605)

---

## 7. Practical Implications for Skill Design

### Language/Framework Recommendations

1. **TypeScript over JavaScript** — 94% of LLM compilation errors are type errors that TypeScript catches automatically. TypeScript consistently outperforms JavaScript in benchmarks.

2. **Opinionated frameworks over flexible ones** — Rails/Next.js/Django produce more consistent AI output than Express/Flask/Fastify. Convention-following is AI's strength.

3. **Type safety is the cheapest quality gate** — It catches errors before tests run, before review happens, before deployment. For AI-generated code, this matters even more than for human code.

4. **Framework-level AI support matters** — Next.js's AGENTS.md approach (100% eval pass rate) shows that frameworks designed for AI agents produce measurably better results. This advantage will compound as more frameworks invest here.

5. **Codebase health is non-negotiable** — Below 7.0 Code Health, AI assistants actively increase defect risk by 30%+. Skills should include code health awareness and refuse to apply AI to severely unhealthy code without guardrails.

### What This Means for Skills

- Skills targeting Python will work best out-of-the-box due to training data concentration
- Skills targeting TypeScript/Java/Rust benefit from compiler-as-verifier patterns
- Skills for JavaScript/Go/C++ need extra verification steps to compensate for lower baseline performance
- All skills benefit from: explicit conventions, type annotations, health-aware guardrails, and tight test-run-fix loops
- Framework-specific context (like Next.js AGENTS.md) dramatically outperforms generic approaches

### The Training Data Bias Problem

The Multi-SWE-bench results reveal a critical issue: AI models appear far more capable than they are because benchmarks concentrate on Python. A model achieving 52% on Python SWE-bench may achieve only 2-8% on TypeScript or Go issues. Skills must account for this reality rather than assuming uniform capability across languages.
