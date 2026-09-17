# Implementation brief template

Write `_docs/heist/brief.md`. This is what The Safecracker implements. An unfamiliar engineer should be able to ship from this file alone.

```markdown
# Brief: <title>

## Problem
What's broken or missing. Who feels it and how.

## Target User
A specific person or role, not a category.

## Core Wedge
Narrowest version still worth building. What it does. What it deliberately leaves out.

## Scope
- In:
- Out:
- Deferred (with why):

## Current State
What the repo does today, with file/symbol citations. Greenfield: say what you searched.

## Proposed Change
The approach, not a tutorial. Name the modules that change. Include a short ASCII diagram for each new data flow (happy / nil / empty / error).

## Acceptance Criteria
- [ ] Checkable true/false statements. "Better errors" is not an AC.

## Out of Scope
Adjacent work the Safecracker must not absorb.

## Constraints
Reuse X. Do not add dependency Y. "None." if empty.

## Edge Cases & Failure Modes
Table or bullets. Unhandled cases are not allowed in a ready brief.

## Test Plan
Maps to the test matrix: what to unit-test, what to integration-test, what to verify by inspection.

## Rollback
How to undo if this ships wrong.

## Assumptions
Taken as given; user should flag if wrong.

## Open Questions
Omit this section if there are none. Unresolved questions that would change the code mean the brief is not ready.
```

Omit empty sections. Do not pad. Implementation-level pseudocode is a smell — specify behavior, not a line-by-line rewrite of the Safecracker's job.
