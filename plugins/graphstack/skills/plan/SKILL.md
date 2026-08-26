---
name: plan
description: Use when the user wants a graphstack brief (or any raw idea/spec, if Think hasn't run yet) turned into a scoped plan and a task backlog before implementation starts. Second stage of the graphstack pipeline — reads `_docs/graphstack/brief.md`, sequences a strategist pass then an architect pass, and writes `plan.md` plus the task backlog. Run before /graphstack:build; run /graphstack:think first if you want a brief written for you, though Plan works without one.
---

# Plan: brief -> scoped plan -> backlog

Turns `_docs/graphstack/brief.md` into `_docs/graphstack/plan.md`, `_docs/graphstack/tasks.md`, and one groomable task file per task under `_docs/graphstack/backlog/`. Load `reference/task-template.md` in this skill directory before writing task files. See `reference/conventions.md` (this plugin's root) for the state-dir rules this and every other stage follow.

## 1. Load prior learnings

If `_docs/graphstack/learnings.md` exists, read it before doing anything else — it's written by the Reflect stage from prior runs. Factor any relevant lessons into scope judgment and architecture decisions below. Skip if it doesn't exist yet.

## 2. Establish the input

Read `_docs/graphstack/brief.md` if it exists — that's the normal path, written by `/graphstack:think`.

If it doesn't exist (Plan invoked with no prior Think run), don't block on that. Fall back to whatever the user gives you directly in this conversation — a pasted idea, a spec, a one-liner. Treat it as the input the rest of this skill works from; note in the eventual hand-off that Plan ran without a brief, so the user knows `/graphstack:think` is available if they want that framing pass later.

## 3. Sequence the strategist and architect passes

Run two agent passes against the input, in order — a strategist pass for scope framing, then an architect pass for architecture lock-in. Don't run them in parallel: the architect needs the strategist's settled scope as input, not a scope that might still move.

1. **Strategist pass.** Dispatch the `strategist` agent with the input (brief path or raw content). It picks a scope mode (Expansion / Selective Expansion / Hold Scope / Reduction), judges premises, and hands back a scope decision — what's in, what's out, what's deferred. If it raised any genuinely close scope calls to the user via `AskUserQuestion`, those are already resolved by the time it hands back.
2. **Incorporate before continuing.** Read the strategist's handback in full before starting the next pass. Don't start the architect pass until the scope decision is settled — an architecture review against scope that might still change is wasted work.
3. **Architect pass.** Dispatch the `architect` agent with the input plus the strategist's scope decision. It locks in component boundaries, data flow (happy/nil/empty/error paths), edge cases, failure modes, and a test matrix scoped to what the strategist approved. It has no `AskUserQuestion` tool — any findings that need a human call (infeasibility, a scope call architecture review destabilized) come back in its handback text instead of an interactive prompt.
4. **Surface only what's genuinely unresolved.** Between the two passes you'll usually have everything you need to write the plan without further back-and-forth. Only go back to the user for scope or architecture calls that are genuinely close or that either agent explicitly flagged as needing a decision — don't manufacture a review gauntlet out of settled judgment calls.

This mirrors how gstack's `autoplan` sequences its CEO-mode and eng-mode reviews: run one pass to completion, fold its output into the next pass's input, then run the second pass — no separate orchestrator process, just ordered delegation within this skill.

## 4. Write the plan

Create `_docs/graphstack/` if it doesn't exist yet. Write `_docs/graphstack/plan.md` combining both passes:

```markdown
# Plan: <title>

## Scope
What's in, what's out, what's deferred, and why. From the strategist pass.

## Architecture
Component boundaries, data flow, edge cases, and failure modes. From the architect pass.

## Test Matrix
What needs unit vs integration/E2E coverage, and the happy/failure/edge cases per area. From the architect pass — this seeds task Acceptance Criteria below.
```

## 5. Decompose into tasks

Break the plan into small, independent tasks — each one shippable and verifiable on its own. Use the architect's test matrix to ground each task's eventual Acceptance Criteria in something concrete rather than vague.

Write `_docs/graphstack/tasks.md` as an index: task slug, one-line title, status column. This is the file `/graphstack:build` reads to pick the next task — keep it in sync with the backlog files.

For each task, create `_docs/graphstack/backlog/<slug>.md` using the template in `reference/task-template.md`, frontmatter `status: open`. Leave grooming (turning it into checkable acceptance criteria) to the Build phase — it's done one task at a time there, so grooming can account for what earlier sibling tasks actually produced. Same lazy-grooming rationale the graph-engineer plugin already uses: don't front-load detail that later context would make more accurate.

## 6. Hand off

Tell the user:
- How many tasks were created and where (`_docs/graphstack/tasks.md`, `_docs/graphstack/backlog/`).
- Where the plan itself lives (`_docs/graphstack/plan.md`).
- Whether Plan ran with a brief or with raw input (if no `/graphstack:think` run preceded this).
- That `/graphstack:build` is next and will start working through the backlog.

## Rules

- Never write or edit application code in this stage — Plan only produces the plan and the backlog.
- Don't skip the strategist pass just because the input looks already well-scoped — Hold Scope is a legitimate outcome of running it, not a reason to skip running it.
- Don't run the architect pass before the strategist pass has handed back a settled scope decision.
- Reuse `reference/task-template.md` verbatim for every backlog file — don't reinvent the shape per task.
