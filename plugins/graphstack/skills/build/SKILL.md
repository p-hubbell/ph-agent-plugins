---
name: build
description: Third stage of the graphstack pipeline. Works through the task backlog created by the plan stage (`_docs/graphstack/tasks.md`) — grooms and implements each task via a Groomer -> Software Engineer loop, stopping once every task reaches `implemented` (or further). Run /graphstack:plan first if no backlog exists yet. Hands off to /graphstack:review next; QA/review are out of scope for this stage.
---

# Build: implement the backlog

You (the orchestrator) drive this loop directly — don't delegate the looping itself to an agent, only the per-task work. Load `reference/backlog-format.md` in this skill directory for the exact status enum and who sets each value before starting.

If `_docs/graphstack/tasks.md` doesn't exist yet, tell the user to run `/graphstack:plan` first instead of improvising a backlog.

This stage is single-responsibility: it takes tasks from `open` to `implemented` and stops there. It never invokes review or QA — those are the separate `/graphstack:review` and `/graphstack:test` stages downstream.

## Loop, per task

1. Read `_docs/graphstack/tasks.md`; pick the next task whose status is not yet `implemented` or further (i.e. status is one of `open`, `groomed`, `in-progress`, `review-failed`, `qa-failed`).
2. Branch on its status:
   - **`open`**: invoke the `groomer` agent with the task file path to groom it. Confirm it came back `groomed` before continuing to the next step.
   - **`groomed`** (freshly groomed above, or already groomed on disk): set the task file's frontmatter `status` to `in-progress`, then invoke the `software-engineer` agent with the task file path.
   - **`in-progress`** (found already in this state, e.g. resuming after an interruption): invoke `software-engineer` directly — no need to re-set the status.
   - **`review-failed`** or **`qa-failed`** (a downstream stage sent this task back for rework): re-invoke `software-engineer` directly with the task file path. It will read the relevant `## Review Log` or `## QA Log` entries itself and re-implement. Do not re-groom — the task's Acceptance Criteria already stand.
3. Confirm `software-engineer` set the task's status to `implemented`. Update `tasks.md`'s index row for this task. Commit if this is a git repo (one commit per task that reaches `implemented` — check `git status`/`git diff` first as usual, don't commit unrelated changes).
4. Report progress on this task (task name, prior status -> `implemented`, whether you committed), then move to the next task.

## Stop conditions

Surface these to the user instead of continuing silently:

- Every task in `tasks.md` is `implemented` or further (`reviewed`, `tested`, `done`) — report the final count and that Build is done.
- `software-engineer` reports (in Implementation Notes) that a task's acceptance criteria are unimplementable or contradictory — stop on that task, report what's blocking it, and ask the user how to proceed rather than guessing.
- The user gave an explicit stop condition when invoking this skill (e.g. "just get the first task done") — honor it over working the full backlog.

## Hand-off

When the loop stops, report:

- How many tasks reached `implemented` this run.
- Any tasks left un-implemented and why (blocked, explicit stop condition, etc).
- That `/graphstack:review` is the next stage to run.

## Rules

- Report progress after each task completes — don't run the whole backlog silently and summarize once at the end.
- Never let `software-engineer` set a task's status to anything but `implemented` — only this loop sets `in-progress`; `reviewed`, `review-failed`, `tested`, `qa-failed`, and `done` belong exclusively to the Review, Test, and Ship stages.
- Never invoke a review or QA agent from this loop — that's out of scope for Build, by design, so Review and Test can each own their own gate.
- Never skip grooming for an `open` task, even if it looks simple enough to implement as-is.
