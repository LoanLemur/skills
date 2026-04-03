# Convention Review Checklist

For use by any sub-agent performing convention/quality review.

## Review Process

1. Run `git diff` (or whatever the calling skill specifies) to see changes
2. **Structural gate (before code review):**
   - List every distinct change in the diff (refactors, features, bug fixes). If more than one category, recommend splitting. More atomic commits is always correct.
   - For every new file added or pattern replaced, grep for references to the old file/pattern. Flag dead code that should be deleted in this commit.
3. **Subtraction pass:** For every added line: "Why can't I delete this?" For every new file: "Does the project already handle this?" For every test: "What confidence does this provide that reading the code does not?" Flag anything that cannot justify its existence. Common patterns: if `field_error_proc` handles errors, manual error divs must be deleted. If a model spec tests the method, the controller spec does not re-test it. If a scope already exists for similar filtering, the controller does not inline the query. If `be_successful` is the only assertion in a controller test, the test must justify why that provides confidence beyond reading the action.
4. **Convention pass:** Evaluate each changed file against the checklist below. Cite the specific CLAUDE.md convention at the moment of evaluation — not in a separate pass afterward.
5. **Checklist gate:** Re-read findings. For every design decision described, verify a specific CLAUDE.md convention was cited. If a decision lacks a citation, evaluate it now. This is mechanical — pattern-match against your own output for the absence of convention citations.

## Convention Evaluation

- **Responsibility** — Does the logic belong where it is? Fat models, skinny controllers. Business logic in models, not controllers. Does the controller export ivars the view can reach through an association on another ivar? For any query construction in a controller beyond a simple `where` (joins, subqueries, FK column references, `merge`), grep the model for existing scopes — if similar scopes exist (e.g., `with_any_tags`), the new query belongs in a model scope.
- **DRY** — Is the same concept implemented in more than one place? Flag as duplication, not correctness.
- **Dependency** — Is this value already available locally? A method reaching into an association or API when the data exists locally creates unnecessary coupling.
- **State safety** — Would a crash leave data inconsistent? Are defaults crash-safe? Does a public method silently depend on another method being called first? Create DB records BEFORE external API calls.
- **Error provenance** — Trace error handling to source. Would changing the error at the source eliminate downstream handlers?
- **Idiom conformance** — Rails built-ins, RESTful controllers, declarative over imperative. Non-CRUD controller actions are suspect.
- **Commit purity** — Does the diff mix a refactor with a feature? One category per commit.

## HTTP Semantics

- GET requests must be safe (no side effects)
- POST/PATCH/PUT/DELETE for mutations only
- REST verbs must match actual behavior
- Idempotency expectations respected

## Security & Tenant Isolation

- Every controller action must declare `authorize`
- Queries scoped to `current_merchant`
- Strong parameters for all user input
- No PII in URLs, query params, or DOM attributes
- Server-side data stays server-side

## Hotwire Idioms

- Prefer Turbo Frames/Streams over Stimulus
- Turbo Frame IDs must match between page and response
- Turbo Streams must target DOM IDs that exist
- Verify frame hierarchy for nested frames

## Convention Conformance in New Files

- For every new view template, compare against existing templates in the same directory. If the new file handles something differently, flag it — the project already has a convention.
- Check `config/initializers/` for global behavior before assuming new code needs to handle cross-cutting concerns manually.

## Review Posture

- State findings as recommendations with the fix, not as open questions.
- Go to the cleanest answer the first time — don't stop at intermediate improvements.
- Leading questions are a cop-out. If you know the answer, state it.
- Don't soften real findings. No "not a big deal," "I wouldn't insist," "it works fine either way."
- If it's worth raising, recommend the fix. If not, don't mention it.

## Test Honesty

- For every `it` block, verify each noun and verb in the description has a corresponding assertion. Flag dishonest descriptions.

## Anti-Narration Bias

The core failure mode is narration — accurately describing what code does without evaluating whether it should. At the moment you describe a design decision, cite the convention it follows or violates. If you find yourself describing without evaluating, stop.

## Output Format

One sentence overall assessment, then findings ordered by severity:

1. **This is wrong** — bugs, data loss, security issues
2. **This violates convention** — CLAUDE.md violations, pattern breaks
3. **This is unnecessary** — code, tests, or abstractions that shouldn't exist
4. **This could be simpler** — minor improvements

For each: file:line reference, the problem, the recommendation, the reasoning. One sentence each. Show code examples for fixes when helpful.

If the code is clean, say so. Do not manufacture findings.
