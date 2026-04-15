## Implementation Plan

### Approach

- {Chosen approach in one line.} {One-line why.}
- Rejected {alternative}. {One-line why.}

{Omit the rejected line only if no alternative was evaluated. Max 3 bullets.}

### Data model

| Table | Columns | Notes |
|-------|---------|-------|
| {name} | {col:type NOT NULL, col:type, fk→other} | {why it exists, one line} |

{Omit section if no schema changes. No narrative.}

### Technical decisions

- **{Decision}.** {Why, one line.}
  Rejected {alt} ({one-line reason}).

- **{Decision}.** {Why, one line.}
  Rejected {alt} ({one-line reason}).

{One blank line between decisions. No "Considered:" preamble.}

### Capabilities

1. {Capability name}
   - Adds: {what ships}
   - Pattern: {existing file/feature to mirror}
   - Nuance: {only if non-obvious — else omit}

2. {Capability name}
   - Adds: {what ships}
   - Pattern: {existing file/feature to mirror}

{Blank line between capabilities. Omit bullets that don't apply. Annotate `[design-identified]` if not in original inventory.}

### Behavior traceability

| Behavior | Capability |
|----------|------------|
| {Behavior 1} | {N} |
| {Behavior 2} | {N, M} |
| {Behavior 3} | Deferred → #{issue} |

### Implementation notes

- {Cross-cutting pattern, one line}
- {State transition: A → B on {trigger}}
- {Calculation: {formula}}
- {Related: #{issue}}

{Omit section if no cross-cutting notes.}

### Testing strategy

- {Capability N}: {specific calculation or edge case}
- {Capability M}: {specific calculation or edge case}

{Omit section if nothing warrants tests. Never write "test the model."}
