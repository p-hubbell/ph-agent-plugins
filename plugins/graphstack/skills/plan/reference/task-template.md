# Task file template

Every file under `_docs/graphstack/backlog/<slug>.md` follows this shape.

```markdown
---
status: open
---

# <Task title>

## Goal

One or two sentences: the outcome this task delivers, not how to build it.

## Acceptance Criteria

- [ ] Checkable statement a QA engineer can verify true/false by inspection or a test run.
- [ ] ...

## Out of Scope

- Anything adjacent this task explicitly does not cover.

## Constraints

- Technical or product constraints that limit the solution space. Omit the section body (leave "None.") if there are none.
```

`status` moves through: `open` -> `groomed` -> `in-progress` -> `implemented` -> `reviewed` -> `tested` -> `done`, or `review-failed`/`qa-failed` (looped back to `implemented` after re-work). See `skills/build/reference/backlog-format.md` for the full table of who sets each value.

Appended by later stages, in order, never removed:

- `## Implementation Notes` — added by software-engineer after implementing.
- `## Review Log` — added by the Review stage, one entry appended per pass, oldest first.
- `## QA Log` — added by qa-engineer, one entry appended per QA pass, oldest first.
