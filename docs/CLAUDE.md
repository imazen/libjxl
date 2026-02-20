# Documentation Project Rules

## Absolute Accuracy

Every claim in a chapter MUST be verified against source code. If you haven't read the source, don't write about it.

- Constants: copy exact values from source (e.g., `kQuantWeightXYB[3][6]` with real numbers)
- Formulas: write the actual math, not prose approximations
- Cost functions: document every multiplier, threshold, and decision branch
- Algorithm steps: match source order, not textbook order
- File references: always include `file.cc:line_number` for verifiability

## Cost Analysis Priority

**Decision trees, cost functions, and multipliers are the primary focus.** For every encoder heuristic:
- What costs are computed? (exact formula with constants)
- What thresholds trigger decisions? (exact values)
- What multipliers adjust behavior? (where they come from, what they scale)
- What's the decision tree? (if/else chain with actual conditions)

## Two-Pass Methodology

### Pass 1: Analysis → `docs/analysis/`
Read source files, save structured notes. These notes are the single source of truth.

### Pass 2: Synthesis → `docs/src/`
Read analysis notes (NOT source), write chapters.

## Analysis Note Format

```
# {Subsystem Name}
## Source Files
## Key Types (fields, invariants)
## Key Functions (signature, algorithm)
## Constants (exact values, what they control)
## Cost Functions & Decision Trees (DETAILED — every multiplier, every branch)
## Algorithm Details (pseudocode with real formulas)
## Dependencies
## Mermaid Diagram Data
## Open Questions
```

## Session Recovery

1. Read this file
2. Read `docs/analysis/PROGRESS.md`
3. Read analysis notes for current topic
4. Continue from where progress tracker says

## Quality Rules

- No placeholder text ("TODO", "TBD", "to be documented")
- No vague descriptions ("uses a sophisticated algorithm")
- No prose that can't be verified against source
- Every Mermaid diagram must render correctly
- Cross-reference between chapters using relative links
