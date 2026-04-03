# Spec Review Checklist

For use by sub-agents reviewing product specs and GitHub issues (not code).
For use by any sub-agent reviewing product specs and GitHub issues.

## Evaluate

- **Behavior inventory completeness** — Is every behavior from the "How
  it works" narrative represented? Is each behavior distinct (not a
  sub-step of another)?
- **Behavior specificity** — Does each behavior name a trigger, state
  change, and outcome? Any vague verbs ("manage", "handle", "process",
  "support", "enable") hiding undecomposed work?
- **Key decisions** — Does each decision state what was chosen, why, and
  what was rejected? Are the rejections reasoned, not hand-waved?
- **Constraints** — Are constraints specific and verifiable? "Must be
  fast" is not a constraint. "API rate limit: 100 req/min" is.
- **Edge cases** — Are failure states defined for each integration point?
  What happens on timeout, partial failure, concurrent access?
- **Scope** — Is this the smallest version that delivers value? What
  could ship separately?
- **Actors** — Is every actor named? Is every actor's flow specific?
- **Standalone test** — Could someone design an implementation from this
  issue alone, without the original conversation?

## Output Format

Findings by severity:

1. **This is missing** — gap that will block design or surface during implementation
2. **This is vague** — described but not specific enough to implement from
3. **This is unnecessary** — scope that wasn't asked for or doesn't deliver value
4. **This could be clearer** — understandable but could be misinterpreted

For each: the problem, the recommendation. One sentence each.
If the spec is solid, say so.
