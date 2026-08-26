# Backlog format (build stage)

`_docs/graphstack/tasks.md` — index table: `slug | title | status`. Update it every time a task's status changes; it's the source of truth for "what's next." See `plugins/graphstack/reference/conventions.md` for the state directory layout and full task template.

`_docs/graphstack/backlog/<slug>.md` — one file per task. Status enum and who sets each value:

| status | set by |
|---|---|
| `open` | Plan stage, when the task is first added to the backlog |
| `groomed` | `groomer` agent, after grooming Goal/Acceptance Criteria/Out of Scope/Constraints |
| `in-progress` | Build orchestrator, right before invoking `software-engineer` |
| `implemented` | `software-engineer`, after implementing (and, if applicable, after re-implementing following a `review-failed`/`qa-failed` rework pass) |
| `reviewed` | Review orchestrator, after a Review PASS verdict |
| `review-failed` | Review orchestrator, after a Review FAIL verdict — loops back to `implemented` once `software-engineer` reworks it |
| `tested` | Test orchestrator, after a Test PASS verdict (implies `reviewed` already happened — Test only runs after Review has passed) |
| `qa-failed` | Test orchestrator, after a Test FAIL verdict — loops back to `implemented` once `software-engineer` reworks it |
| `done` | Ship stage, after successfully shipping — terminal |

Only the orchestrator driving the relevant stage's skill sets these statuses. Agents (`groomer`, `software-engineer`, and any Review/Test/Ship-stage agents) only ever move a task at most one step forward from where they found it — same rule enforced across every stage in this plugin.

## What the Build stage does and does not set

The Build stage (`/graphstack:build`) only ever drives a task from `open` through `implemented`:

- Its orchestrator sets `in-progress`.
- Its `groomer` agent sets `groomed`.
- Its `software-engineer` agent sets `implemented`.

The Build stage **never** sets `reviewed`, `review-failed`, `tested`, `qa-failed`, or `done` — those belong exclusively to the Review, Test, and Ship stages. If Build encounters a task already at `review-failed` or `qa-failed` (e.g. re-run after a downstream failure), it may re-invoke `software-engineer` to rework it back to `implemented`, but it does not run review or QA itself to verify that rework — that's the Review/Test stages' job on their next pass.

## Appended sections

Appended by later stages, in order, never removed once present (per `conventions.md`):

- `## Implementation Notes` — added by the Build stage's `software-engineer` agent.
- `## Review Log` — added by the Review stage, one entry per pass.
- `## QA Log` — added by the Test stage's qa-engineer agent, one entry per pass.

`software-engineer` reads `## Review Log` or `## QA Log` (whichever exists) when re-invoked on a task found at `review-failed` or `qa-failed`, but never writes to either section — those stay owned by the Review and Test stages.
