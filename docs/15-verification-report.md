# Verification Report: Agentic Development Research

**Date:** 2026-04-03
**Reviewer:** Verification agent with fresh eyes (no involvement in original research)
**Documents reviewed:** 00-synthesis.md, 01 through 13 (14 does not exist)

---

## Check 1: Internal Consistency

### PASS

- **Pipeline convergence is consistent.** Documents 01, 02, 03, 05, and 07 all describe the same Specify->Plan->Implement->Review pipeline independently. The synthesis accurately represents this.
- **TDD consensus is consistent.** Documents 03, 04, 05 all independently cite TDD as the top quality practice. The synthesis correctly reflects this.
- **Context management guidance is consistent.** The "60% degradation threshold," "/clear between tasks," and "200-line CLAUDE.md budget" appear in 03, 04, 05, 06, and the synthesis without contradiction.
- **Anti-patterns align.** Document 06 catalogs failure modes; 04 covers them from the QA angle; the synthesis table in Part 4 matches both.

### WARN

- **Code Health score varies.** Document 05 (Tornhill section) says "at least 9.5 (ideally 10.0)." Document 06 (section 8, item 6) says "at least 9.4/10." Document 07 says "9.4/10." Document 11 says "9.5 out of 10." The synthesis uses "9.4/10" in the data table. The original CodeScene sources likely say 9.5 for AI (vs 9.0 for humans). This is a minor discrepancy but the synthesis should pick one number and cite it precisely.

- **Security failure rate.** The synthesis table says "45%." Document 04 says "29-45%" in one place and "45%" in another. Document 05 says "55% were secure" (implying 45% insecure). Document 07 says "45%." These are consistent but the range from 04 (29-45% in the intro vs 45% in Veracode data) could confuse a reader.

- **"81% quality improvement" claim.** The synthesis says "81% quality improvements." Document 04 says "81% report quality improvements vs 55% without review." This is a survey result (81% of developers *report* improvement), not a measured 81% *quality improvement*. The synthesis rounds this into a misleading shorthand. An agent reading "81% quality improvements" would interpret it as an objective metric.

### FAIL

- **Hook handler types miscounted in synthesis.** The synthesis (Part 3, Cross-Cutting Concerns) lists hooks and implies 3 handler types. Document 10 (Anthropic Official Docs) documents **4 handler types**: command, http, prompt, and agent. The synthesis never mentions `http` or `prompt` handler types. This is a factual error an agent would carry forward.

- **Hook event count wrong.** The synthesis mentions "24 events" (from doc 01). Document 10 documents **26 events**. The synthesis is stale relative to the later research.

- **Synthesis claims "01-07" as its research base but documents 08-13 exist.** The synthesis header says "Full citations in supporting documents 01-07" and the source table lists only 01-07. Documents 08 (Agent Skills Standard), 09 (Source Leak Insights), 10 (Anthropic Official Docs), 11 (Language/Framework Impact), 12 (Missed Practitioners), and 13 (Self-Improving Skills) are not referenced. **The synthesis was never updated after the second wave of research.** This is the most significant structural failure.

---

## Check 2: Completeness Against Original Brief

| Requested Topic | Coverage | Assessment |
|---|---|---|
| How to get the most out of agentic coding (not vibe coding) | Thoroughly covered across all documents | **PASS** |
| How a senior system architect would use it | Covered in 05 (community wisdom), 07 (enterprise), synthesis P9 | **PASS** |
| Current state of the art | Doc 01 surveys 12 tools as of April 2026 | **PASS** |
| Claude Code specifically | Deep coverage in 01, 03, 04, 05, 09, 10 | **PASS** |
| OpenAI Codex specifically | Covered in 01, 07 (section 5) | **PASS** |
| github.com/SinghCoder/claude-code (open-source port) | **NOT COVERED ANYWHERE.** No document mentions this repository. Doc 09 discusses the source leak and mentions "claw-code" (the clean-room rewrite that hit 75K+ stars) but never mentions SinghCoder/claude-code. | **FAIL** |
| Kiro | Good coverage in 01 (tool landscape), 02 (spec-driven dev), 13 (section 10) | **PASS** |
| Anti-Gravity (Google Antigravity) | Covered in 01 and 02 | **PASS** |
| Skills designed to create exceptional output | Covered in 08 (Agent Skills Standard), synthesis Part 3, doc 10 (skills spec) | **PASS** |
| Best practices from experienced/exceptional teams | Strong coverage in 05 (community wisdom), 07 (enterprise), 12 (missed practitioners) | **PASS** |
| Pipeline for feature development | Core topic of synthesis Part 2, doc 02, doc 03 | **PASS** |

### FAIL: github.com/SinghCoder/claude-code

This was explicitly requested by the user. The user even previously caught the research missing the source code leak topic. Despite that correction, the specific open-source port at github.com/SinghCoder/claude-code was still never researched. This is the exact failure mode the user warned about. Document 09 discusses the leak and the claw-code rewrite but never mentions SinghCoder's repository by name or examines its approach.

---

## Check 3: Source Quality

### PASS

- **Named sources with URLs are pervasive.** Every document has a Sources section with clickable links. The research is well-sourced overall.
- **Major institutional sources are credible:** Anthropic official docs, Martin Fowler/Thoughtworks, METR (academic study), Veracode (security vendor), DORA (Google), CodeScene (peer-reviewed research), GitHub (2500-repo analysis), Stripe Engineering blog.
- **Practitioner sources are named and linked:** Simon Willison, Kent Beck, Addy Osmani, Boris Cherny, Harper Reed, Ran Isenberg, incident.io, Stavros Korokithakis.

### WARN: Potentially Hallucinated or Unverifiable Claims

- **"Spec Kit 80K+ stars" (doc 03).** This claim for GitHub's spec-kit seems extremely high for a dev tool spec framework. GitHub's own spec-kit repo should be verifiable, but 80K stars for a spec-driven development toolkit released in 2025 is suspicious. For comparison, the actual claude-code repo has 82K stars. This deserves verification.

- **"anthropics/skills repository hits 110k stars" (doc 08).** This seems plausible given the source leak drove traffic, but is a very high number. The document does provide a source URL.

- **"claw-code reached 75,000+ stars -- fastest repo to 50K stars in GitHub history" (doc 12).** Plausible but extraordinary claim. Sourced to Layer5 blog.

- **Google Antigravity announced "November 2025 alongside Gemini 3" (doc 02).** Other references say it uses "Gemini 3.1 Pro" (doc 01). These could be consistent (announced with Gemini 3, later upgraded), but the specificity of the November 2025 date and the naming of "Gemini 3" should be verified. The model names (Gemini 3, Gemini 3.1 Pro) are plausible for a rapidly iterating product.

- **"Boris Cherny (Staff Engineer at Anthropic, creator of Claude Code)" (doc 05).** Doc 09 quotes him confirming the leak was "plain developer error." The title is consistent across sources. Verified by cross-reference.

- **Claude Opus 4.6 / Sonnet 4.6 model names.** These appear throughout and are consistent with each other. However, current knowledge (April 2026) should be checked. The internal codenames from doc 12 (Capybara = Claude 4.6, Fennec = Opus 4.6) are from the leak analysis and should be treated as unverified.

- **"DORA 2025" and "DORA data shows elite-performing teams are 'pretty much all doing TDD'" (synthesis P3, doc 04).** The DORA quote appears in doc 04 attributed to Codemanship (Jan 2026), not directly to the DORA report. The synthesis implies it comes from DORA directly. This is a sourcing error -- the claim is second-hand.

### FAIL

- **METR study interpretation.** All documents consistently report "19% slower" from the METR study. However, document 06 says the study ran "Feb-Jun 2025" while document 05 says METR "ran a randomized controlled trial with 16 experienced open-source developers on 246 real issues." These are consistent. The synthesis correctly notes METR is "redesigning their experiment" (doc 05). No factual error found, but the small sample size (16 developers) and the study's own acknowledged limitations should be more prominently flagged in the synthesis. The synthesis presents "19% slower" as a key data point without the caveats.

---

## Check 4: Staleness

### WARN

- **The synthesis was written before documents 08-13 existed.** The synthesis references "100+ sources across... Full citations in supporting documents 01-07." It was never updated to incorporate the second wave of research (08-13). This makes the synthesis stale by definition.

- **Kiro description may be stale.** Doc 01 says Kiro is "GA since November 2025" with an "autonomous agent in preview." Doc 02 references Kiro's features. Doc 13 says knowledge management is a feature request (Issue #6988), suggesting it's still being developed. This is internally consistent for April 2026.

- **METR study.** The study is from Feb-Jun 2025, with METR redesigning as of Feb 2026. No newer results are cited. The synthesis correctly identifies this as the "most rigorous" study. Given the acknowledged limitations and the one-year age, a stronger caveat is warranted.

### PASS

- **Tool landscape (doc 01) is dated April 2026** -- current.
- **Anthropic 2026 Trends Report** data is from Q1 2026 -- current.
- **Veracode Spring 2026 Update** is referenced -- current.
- **Source leak coverage (doc 09) is from March 31, 2026** -- very current.

---

## Check 5: The Synthesis as a Skill-Authoring Spec

### Could an agent read ONLY 00-synthesis.md and produce quality skills?

**No.** The synthesis provides good principles and a pipeline, but critical implementation details are missing.

### FAIL: Agent Skills Standard Not Documented

The synthesis says "Skills follow the Agent Skills open standard" (Part 3, Principle 2) but never documents what that standard is. An agent reading only the synthesis would not know:
- The SKILL.md file format
- Required vs optional frontmatter fields
- The progressive disclosure model (metadata -> instructions -> resources)
- Directory structure conventions (scripts/, references/, assets/)
- Name constraints (lowercase, hyphens, must match directory name)
- The 500-line recommendation for SKILL.md
- Description length limits (250 chars)

Document 08 provides all of this. The synthesis should either include it or explicitly reference doc 08 with a "read this before authoring skills" directive.

### FAIL: Sub-Agent Execution Models Not Documented

The synthesis mentions sub-agents as a context management tool but never documents:
- The three execution models (Fork, Teammate, Worktree) from doc 09
- When to use each model
- The subagent file format (`.claude/agents/*.md`)
- Frontmatter fields for subagents
- The architectural constraint that subagents cannot spawn subagents
- The `context: fork` mechanism for skills

An agent trying to design skills that dispatch sub-agents would have to guess at the implementation.

### FAIL: Language/Framework Considerations Absent

The synthesis contains no language or framework-specific guidance. Document 11 reveals:
- 24x performance gap between Python and TypeScript on Multi-SWE-bench
- 94% of LLM compilation errors are type-related (caught by TypeScript)
- Java has a 70%+ security failure rate in AI-generated code
- Opinionated frameworks (Next.js, NestJS, Rails) produce significantly better AI output than flexible ones (Express, Flask)
- Codebase health must be 9.5/10 for reliable AI performance

None of this appears in the synthesis. An agent authoring skills without this knowledge would not include language-specific verification steps or framework-aware guidance.

### FAIL: Self-Improving/Self-Correcting Skill Design Not Addressed

The synthesis has zero content on:
- How skills should evolve based on failures
- The Boris Cherny "compounding engineering" pattern
- Hook-based learning systems
- The reflection prompt pattern
- Configuration drift risks
- The manual -> semi-automated -> automated spectrum

Document 13 covers this comprehensively. A skill-authoring agent would produce static skills with no feedback mechanism.

### WARN: Plugin Ecosystem Not Mentioned

Document 10 reveals an entire plugin distribution system (marketplace, namespaced skills, LSP servers, binary distribution). The synthesis doesn't mention plugins at all. An agent authoring skills would not know they can be packaged as distributable plugins.

### WARN: Key Claude Code Features Missing

An agent reading only the synthesis would not know about:
- `/btw` for side questions without context growth
- `/rewind` and the checkpoint system
- The auto mode classifier (sophisticated layered safety, not just "skip permissions")
- Agent teams (experimental multi-agent coordination)
- `@import` syntax in CLAUDE.md
- Skill `!`backtick`` preprocessing for dynamic context injection
- The `context: fork` mechanism
- `disable-model-invocation` and `user-invocable` frontmatter fields

### PASS

- **The 10 principles are sound.** P1-P10 are well-supported by the research and internally consistent.
- **The pipeline (Part 2) is actionable.** The four phases with inputs/outputs/gates would guide an agent.
- **The anti-pattern table (Part 4) is comprehensive** for the first wave of research.
- **The data table (Part 5) is useful** for calibrating expectations.
- **Skill descriptions in Part 3 are clear** about when/what/output/rigidity.

---

## Check 6: Contradictions with Known Facts

### WARN

- **"Compound accuracy: 85%/step, 10 steps = ~20%"** -- The math checks out (0.85^10 = 0.197). Sourced to doc 06 which cites DEV Community. The 85% per-step figure is an assumption, not a measured value. The synthesis presents it as a "key data point" when it's really a back-of-envelope calculation.

- **"CLAUDE.md instruction budget: 150-200 lines"** -- Doc 04 says "~150-200 instruction budget before compliance drops" and "system prompt uses ~50 of those." Doc 07 says "Configuration files degrade agent performance above 150 lines." These are consistent in direction but the 150-200 range is soft. No one has published a rigorous study measuring compliance vs. CLAUDE.md length. The synthesis presents it as harder science than it is.

- **"Atomic vs bundled task cost: 59% less"** -- Sourced to a single blog post (doc 03) comparing one task done two ways. This is an anecdote, not a study. The synthesis elevates it to a "key data point."

### PASS

- **METR "19% slower"** -- Consistently cited, properly caveated in supporting docs.
- **"78% multi-file sessions"** -- From Anthropic's own trends report. Credible first-party data.
- **"25.8% shipping without review"** -- From Qodo survey. Properly attributed.
- **Veracode security data** -- Consistent across docs 04, 05, 06, 07, 11. Multiple report dates cited (2025, Spring 2026).

---

## Summary of Findings

### Critical Issues (FAIL)

1. **Synthesis never updated for docs 08-13.** The synthesis was written after the first 7 research documents. Six additional documents were produced covering the Agent Skills standard, source leak architecture, official Claude Code docs, language/framework impact, additional practitioners, and self-improving skills. None of this content was incorporated into the synthesis. This is the single biggest problem.

2. **github.com/SinghCoder/claude-code never researched.** Explicitly requested by the user. Not covered in any document.

3. **Agent Skills standard not documented in synthesis.** Synthesis says "follow the standard" without explaining what it is.

4. **Sub-agent execution models not documented.** Fork, Teammate, Worktree models are critical for skill design but absent from synthesis.

5. **Language/framework guidance absent.** 24x performance gaps across languages are relevant to skill design but not in synthesis.

6. **Self-improving skill design not addressed.** Entire document 13 worth of content missing from synthesis.

7. **Hook event count and handler types wrong.** Synthesis says 24 events / implies 3 handler types. Actual: 26 events / 4 handler types.

8. **"81% quality improvements" is misrepresented.** This is a survey result (81% of developers *report* improvement), not a measured quality improvement metric.

### Warnings

1. Code Health score varies between 9.4 and 9.5 across documents.
2. Spec Kit "80K+ stars" claim is suspicious and should be verified.
3. DORA/TDD quote is second-hand (via Codemanship), not directly from DORA.
4. "59% less" atomic task cost is a single anecdote, not a study.
5. "85% per-step accuracy" is an assumption, not a measurement.
6. METR study caveats (n=16, acknowledged limitations) should be more prominent.
7. Plugin ecosystem not mentioned in synthesis.
8. Multiple Claude Code features missing from synthesis (btw, rewind, auto mode classifier, agent teams).

### Passes

1. Internal consistency across docs 01-07 is strong.
2. Pipeline description is accurate and well-supported.
3. Principles P1-P10 are sound and multiply sourced.
4. Sources are named, linked, and generally credible.
5. Tool landscape coverage is current as of April 2026.
6. Anti-pattern catalog is comprehensive.
7. Coverage of requested topics is complete except for SinghCoder/claude-code.

---

## RECOMMEND: Specific Changes to the Synthesis

### Must Do

1. **Rewrite the synthesis to incorporate docs 08-13.** The current synthesis is based on half the research. At minimum:
   - Add the Agent Skills standard specification (from doc 08) as a new section or appendix
   - Add sub-agent execution models (Fork/Teammate/Worktree from doc 09)
   - Add language/framework considerations (from doc 11)
   - Add self-improving skill design patterns (from doc 13)
   - Update hook counts to 26 events / 4 handler types (from doc 10)
   - Add Claude Code features critical for skill design: `context: fork`, `disable-model-invocation`, `!`backtick`` preprocessing, plugin ecosystem (from doc 10)

2. **Research github.com/SinghCoder/claude-code.** This was explicitly requested. Create a document (14) covering it, or explain why it was excluded.

3. **Fix the "81% quality improvements" claim.** Change to: "81% of developers using AI with rigorous review report quality improvements (vs 55% without)." The current phrasing is misleading.

4. **Fix hook handler types.** Change from 3 to 4 (add `http` and `prompt` types). Update event count from 24 to 26.

5. **Settle the Code Health score.** Pick 9.5/10 (the value CodeScene actually publishes) and use it consistently.

### Should Do

6. **Add a "Reading Order" section** to the synthesis directing agents to specific supporting docs for implementation details: "Read doc 08 before authoring any skill. Read doc 09 before designing parallel execution. Read doc 11 if targeting a specific language/framework."

7. **Add language-specific warnings.** At minimum: "AI performance varies 5-24x across languages. Python benchmarks overstate capability for other languages. TypeScript catches 94% of LLM compilation errors at compile time. Java has 70%+ security failure rates in AI-generated code."

8. **Add a self-improvement section.** Cover the Boris Cherny "compounding engineering" pattern, the reflection prompt, and the maturity spectrum (manual -> semi-automated -> automated).

9. **Add caveats to soft data points.** The "59% cost reduction" and "85% per-step accuracy" should be marked as illustrative examples, not empirical findings.

10. **Add the plugin/distribution model.** Skills can be distributed as plugins with namespace isolation, marketplace discovery, and LSP server bundling.

### Nice to Have

11. Mention `/btw` and `/rewind` in context management guidance.
12. Document the auto mode classifier properly (it's a separate Sonnet model, not just "skip permissions").
13. Add the "gold standard file" pattern from Stack Overflow (doc 12) to the CLAUDE.md guidance.
14. Add the "run identical tasks multiple times" pattern from MIT Missing Semester (doc 12).
15. Add the "quality inversion" concept from Apiiro (doc 12) -- AI fixes shallow problems while introducing deep architectural flaws.
