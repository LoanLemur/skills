# Named Personas vs Plain Sub-Agents: Does the Persona Add Value?

Research question: In agentic coding tools like Claude Code, when should you use a named custom agent with a persona (e.g., "DHH", "Devil's Advocate") versus a plain sub-agent with just a task description?

## TL;DR

The persona is mostly a vehicle for the checklist. Recent research (March 2026) shows that expert persona prompting **hurts accuracy on knowledge-retrieval and coding tasks** while helping only on alignment/style tasks. The real value in your DHH and Devil agents comes from three things: (1) the detailed review checklists they carry, (2) context isolation from the parent conversation, and (3) the constrained posture instructions ("state findings as recommendations, not observations"). The name "DHH" is a convenient label, but a nameless sub-agent with the same checklist and posture instructions would produce equivalent or better output.

## 1. What the Research Says About Persona Prompting

### The PRISM Paper (March 2026, USC)

The most rigorous recent study: "Expert Personas Improve LLM Alignment but Damage Accuracy" ([arXiv:2603.18507](https://arxiv.org/html/2603.18507)).

Key findings:
- **Coding tasks are hurt by personas.** On MT-Bench's Coding category, expert persona prompting caused a -0.65 point decline. The authors attribute this to coding depending on "strict zero-shot logical chains" and "precise retrieval of pretrained knowledge."
- **Alignment tasks are helped.** Writing, extraction, and STEM improved with personas. Safety alignment improved dramatically (+17.7% on JailbreakBench).
- **The mechanism:** Persona prefixes activate instruction-following mode at the expense of factual recall. "Telling a model it's an expert in a field does not actually impart any expertise" -- it hinders fact retrieval from pretraining data.
- **MMLU accuracy dropped** from 71.6% baseline to 68.0% with minimal persona and 66.3% with long persona. Every persona variant was worse than no persona.

### The EMNLP 2024 Paper

"When 'A Helpful Assistant' Is Not Really Helpful" ([arXiv:2311.10054](https://arxiv.org/abs/2311.10054v3)) -- 162 roles, 2,410 factual questions, 4 LLM families.

Finding: "Prompting with personas has no or small negative effects on model performance compared with the control setting where no persona is added, consistent across four popular LLM families."

The one exception: if you could magically pick the *right* persona per question, accuracy improves. But automatically identifying the right persona performs no better than random selection.

### Meta's Structured Prompting (2026)

Meta's "semi-formal reasoning" technique achieved 93% accuracy on code review by equipping agents with **structured reasoning templates** -- not personas. Agents must explicitly state premises, trace execution paths, and derive conclusions from evidence. This is essentially a checklist with enforced reasoning structure, and it dramatically outperformed persona-based approaches.

### Summary of Evidence

| Approach | Effect on Code Tasks | Effect on Style/Alignment |
|---|---|---|
| Expert persona ("You are DHH") | Negative (-0.65 on coding) | Positive |
| Task description + checklist | Neutral to positive | Positive |
| Structured reasoning template | Strong positive (93% accuracy) | Positive |
| No prompt engineering | Baseline | Baseline |

**Implication for your agents:** The persona framing ("You are a senior Rails code reviewer") likely provides no benefit for code review accuracy and may slightly degrade it. The checklist and posture instructions are doing the work.

## 2. Anatomy of Your Agents: What's Actually Doing the Work

### DHH Agent (88 lines)

Breaking down the content by function:

| Lines | Content | Category |
|---|---|---|
| 7-9 | "You are a senior Rails code reviewer. Every character is liability..." | **Persona + posture** |
| 11-12 | "Apply the project's design philosophy, 4Cs, testing philosophy..." | **Context pointer** |
| 14-22 | Review process (git diff, structural gate, subtraction pass, convention pass, checklist gate) | **Workflow** |
| 24-45 | Convention evaluation checklist (responsibility, DRY, dependency, state safety, error provenance, idiom, commit purity) | **Checklist** |
| 37-46 | Anti-narration bias section | **Posture** |
| 48-54 | Review posture rules | **Posture** |
| 56-86 | What to check (HTTP, Hotwire, security, conventions, test honesty) + output format | **Checklist + format** |

The persona is ~3 lines. The checklist is ~60 lines. The posture instructions ("state findings as recommendations, not observations", "don't soften real findings") are ~15 lines.

### Devil Agent (73 lines)

| Lines | Content | Category |
|---|---|---|
| 7-8 | "You are a devil's advocate. Your job is to find what's wrong..." | **Persona + posture** |
| 10-16 | Posture rules (state problems directly, prioritize by pain) | **Posture** |
| 18-63 | Challenge categories (scope, assumptions, complexity, completeness, contradictions, failure modes) | **Checklist** |
| 55-63 | When reviewing code | **Checklist** |
| 65-73 | Output format (severity ordering) | **Format** |

Same pattern. The persona is 2 lines. The adversarial checklist is the substance.

### What Would Change If You Removed the Persona?

Replace "You are a senior Rails code reviewer" with "Review this code for Rails convention violations using the following checklist" and "You are a devil's advocate" with "Challenge this design for missing cases, overcomplexity, and incorrect assumptions using the following checklist."

Based on the research, expected result: equivalent or slightly better accuracy on the code review task, because you're not activating the persona-mode attention competition that degrades factual retrieval.

## 3. What IS Providing Value

### Context Isolation (the big win)

Your `/execute` skill spawns 6 sub-agent reviews per implementation (3 per-commit passes x N commits + 3 final passes). Each runs in its own context window. This is the primary architectural benefit -- it prevents review noise from polluting the implementation context.

Evidence: The HAMY Labs pattern (9 parallel nameless sub-agents, ~75% useful findings) achieves strong results purely through context isolation and specialized checklists, with no persona prompting.

### The Checklist (the second big win)

Your DHH agent carries a 7-item convention evaluation checklist (responsibility, DRY, dependency, state safety, error provenance, idiom conformance, commit purity) plus domain-specific checks (HTTP semantics, Hotwire idioms, security, test honesty). This is a structured reasoning template in the Meta mold.

The Devil agent carries a 6-category challenge framework (scope, assumptions, complexity, completeness, contradictions, failure modes).

These checklists constrain the model's attention and force systematic coverage. Without them, a generic "review this code" prompt would produce shallower, more random coverage.

### Posture Instructions (the third win)

"State findings as recommendations, not observations" and "don't soften real findings" are alignment instructions -- exactly the category where the PRISM paper shows persona prompting helps. But note: these are behavioral constraints, not persona identity. "Always state findings as direct recommendations with specific fixes" works without "You are DHH."

The anti-narration bias instruction ("at the moment you describe a design decision, cite the CLAUDE.md convention it follows or violates") is a structured reasoning template. This is the most valuable single instruction in the DHH agent.

## 4. The Three-Pass Pattern: Persona or Architecture?

Your `/execute` skill uses DHH -> Devil -> DHH. The question is whether the value comes from alternating *personas* or alternating *checklists*.

Evidence points to checklists. The three passes work because:

1. **Pass 1 (convention review):** Applies the convention checklist to the diff
2. **Pass 2 (adversarial review):** Applies a different checklist (scope, assumptions, failure modes) to the same diff
3. **Pass 3 (synthesis):** Has findings from passes 1-2 as additional context, applies the convention checklist again with fresh eyes

The value is: (a) two different checklists catching different classes of issues, and (b) a third pass that synthesizes prior findings. Replacing "DHH" with "convention-reviewer" and "Devil" with "adversarial-reviewer" would produce the same behavior, because the model routes on the description field and the system prompt content, not the name.

## 5. The Practitioner Landscape

### Patterns That Use Personas

- **BMAD Method:** Product Manager, Architect, Developer, QA, Scrum Master personas. Each carries a detailed role description and checklist. The personas help with workflow coordination (knowing which agent to invoke when) more than with output quality.
- **Nicholas Zakas (humanwhocodes.com):** Uses personas for AI-assisted development. His argument is that personas provide a mental model for the developer to know what to expect from each agent.

### Patterns That Skip Personas

- **HAMY Labs:** 9 parallel nameless sub-agents (Test Runner, Linter, Security Reviewer, etc.). Each has a task description and output format. ~75% useful findings. No personas.
- **Korokithakis:** Architect (Opus), Developer (Sonnet), Reviewers (cross-vendor). Roles are defined by task, not persona. Uses different vendors for diversity of perspective.
- **"201 Personas, I Installed Zero" (March 2026):** Argues that skills and workflows are reliable and repeatable; personas are not. "A workflow and a persona differ -- one is reliable. The other depends on you remembering to use it."

### The Emerging Consensus

The practitioners getting the best results use **specialized checklists** + **context isolation** + **model diversity** (different models for different passes). Personas are an optional ergonomic layer on top.

## 6. Cost Analysis

Your `/execute` invokes sub-agents roughly:
- 3 passes per commit (DHH, Devil, DHH)
- 3 final passes (DHH, Devil, DHH)
- For a 5-commit feature: 18 sub-agent invocations

Each invocation loads the full agent prompt (~80-90 lines of system prompt). At Opus pricing ($5/$25 per million input/output tokens), this is expensive. At Haiku pricing ($1/$5), it's roughly 5x cheaper.

### Cost Optimization Options

1. **Model selection:** Your agents don't specify a model, so they inherit the parent conversation's model (likely Opus). Code review is a checklist-application task -- Haiku 4.5 reportedly delivers 90% of Sonnet 4.5's agentic coding performance. Setting `model: haiku` on both agents could cut sub-agent costs by ~80% with minimal quality loss for structured review tasks.

2. **Reduce passes from 3 to 2:** The third DHH pass (fresh eyes with prior findings) may not justify the cost. Consider DHH + Devil as two passes, with the Devil agent receiving DHH's findings as context. The synthesis happens in the parent conversation when it decides which findings to act on.

3. **Batch final review:** Instead of 3 final passes, run a single sub-agent with a combined checklist focused on cross-cutting concerns.

4. **Skip review on trivial commits:** Migration-only commits, config changes, and single-line fixes don't need three-pass review.

## 7. Recommendations

### Keep

- **The agents as `.claude/agents/` files.** The file-based approach gives you tool scoping, model selection, persistent memory, and reusable configuration. These are real architectural benefits over inline sub-agents.
- **The checklists.** These are the most valuable part. The convention evaluation checklist and the adversarial challenge framework are structured reasoning templates that produce consistent, systematic reviews.
- **The posture instructions.** "State findings as recommendations" and the anti-narration bias instruction are genuinely valuable alignment constraints.
- **Context isolation via sub-agents.** The three-pass architecture is sound.

### Change

- **Remove persona identity framing.** Replace "You are a senior Rails code reviewer" with a direct task description: "Review the following code changes for Rails convention violations. Apply each checklist item at the moment you encounter a relevant design decision." This avoids the persona-induced accuracy penalty documented in the PRISM paper.
- **Add `model: haiku` to both agents.** Structured review against a checklist is exactly the task profile where Haiku performs well relative to cost.
- **Rename from persona to function.** `convention-reviewer.md` and `adversarial-reviewer.md` communicate what the agent does without implying persona identity. The description field (which Claude uses for delegation) already describes the function, not the persona.
- **Consider reducing to 2 passes per commit** (convention + adversarial). Use 3 passes only for commits that touch security, money, or state transitions.

### The Bottom Line

The persona is a label. The checklist is the mechanism. The context isolation is the architecture. Invest in sharpening the checklists and optimizing the model/pass count rather than refining the persona framing. If you renamed `dhh.md` to `convention-reviewer.md`, changed the opening line from "You are a senior Rails code reviewer" to "Review this diff against the following convention checklist", and set `model: haiku` -- you would likely get equivalent review quality at a fraction of the cost.

## Sources

- [Expert Personas Improve LLM Alignment but Damage Accuracy (PRISM)](https://arxiv.org/html/2603.18507) -- USC, March 2026
- [Telling an AI model that it's an expert makes it worse](https://www.theregister.com/2026/03/24/ai_models_persona_prompting/) -- The Register, March 2026
- [When "A Helpful Assistant" Is Not Really Helpful](https://arxiv.org/abs/2311.10054v3) -- EMNLP 2024
- [Prompting Science Report 4: Playing Pretend](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5879722) -- Mollick et al.
- [9 Parallel AI Agents That Review My Code](https://hamy.xyz/blog/2026-02_code-reviews-claude-subagents) -- HAMY Labs
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents) -- Anthropic official docs
- [Meta's structured prompting for code review](https://venturebeat.com/orchestration/metas-new-structured-prompting-technique-makes-llms-significantly-better-at) -- VentureBeat
- [Multi-Agent Collaboration: 4 Architecture Patterns](https://eastondev.com/blog/en/posts/ai/20260325-multi-agent-system/) -- BetterLink Blog
- [Skill context:fork vs Sub-agent skills](https://zenn.dev/trust_delta/articles/claude-code-skills-subagents-approaches?locale=en)
- [201 Claude Code Agent Personas, I Installed Zero](https://levelup.gitconnected.com/201-claude-code-agent-personas-i-installed-zero-b45e4629512d) -- Level Up Coding, March 2026
- [Role Prompting Guide](https://learnprompting.org/docs/advanced/zero_shot/role_prompting) -- Learn Prompting
- [BMAD Method](https://github.com/bmad-code-org/BMAD-METHOD)
- [How one developer uses multi-agent LLM workflows](https://agent-wars.com/news/2026-03-16-how-one-developer-uses-multi-agent-llm-workflows-architect-developer-reviewers) -- Korokithakis
