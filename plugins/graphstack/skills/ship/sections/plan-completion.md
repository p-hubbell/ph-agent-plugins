# Plan completion audit

This is informational, not gating. Every task in scope already passed Review and Test (Step 1 of the parent skill verified this). This audit's job is to check the shipped diff against the task's own Acceptance Criteria and against `_docs/graphstack/plan.md`'s scope — not to re-litigate whether the code works, but to catch drift: did the diff quietly do more, or less, than what was planned?

If `_docs/graphstack/plan.md` doesn't exist: skip this audit entirely and note "No plan.md found — skipped." in the PR body.

**Dispatch as a subagent** (`general-purpose`). Give it:

> Audit plan completion for the tasks being shipped in this repo. Base branch: `<base>`. Run `git diff <base>...HEAD` and `git log <base>..HEAD --oneline` as needed. Do not commit or push — report only.
>
> 1. Read `_docs/graphstack/plan.md` for the overall scope/architecture this branch is meant to deliver.
> 2. For each task in scope (paths under `_docs/graphstack/backlog/<slug>.md`, passed to you by the caller), read its `## Acceptance Criteria` checklist and its `## Implementation Notes` / `## QA Log` entries.
> 3. For each acceptance criterion, classify against the diff:
>    - **DONE** — clear evidence in the diff (cite the file/lines).
>    - **CHANGED** — implemented via a different approach than the plan implied, but the same goal is met. Note the difference.
>    - **NOT DONE** — the diff shows no evidence this criterion was addressed.
>    - **UNVERIFIABLE** — can't be confirmed or denied from the diff alone (e.g. it depends on external state). Be honest here rather than guessing DONE.
> 4. Separately, check for **scope drift**: files changed that have nothing to do with any task's stated goal, or functionality added that no task's Acceptance Criteria called for. Note it plainly if found — this is informational, not an accusation.
> 5. Output a short checklist per task plus a scope-drift line: `Scope: CLEAN` or `Scope: DRIFT DETECTED — <one-line description>`.
>
> Be conservative with DONE — a file being touched isn't enough, the specific criterion has to actually be met. Be honest with UNVERIFIABLE — better to flag it than silently mark it DONE.
>
> End your output with a single JSON object on the last line: `{"total_items": N, "done": N, "changed": N, "not_done": N, "unverifiable": N, "summary": "<markdown checklist + scope line for the PR body>"}`.

**Parent processing:**
1. Parse the last line as JSON. Embed `summary` verbatim as the PR body's Plan Completion section.
2. If `not_done > 0`: this doesn't block shipping (Review/Test already passed), but tell the user directly — a task passing QA and still having an unaddressed acceptance criterion the audit caught is worth a second look before merge. Name the specific criterion and task.
3. If `unverifiable > 0`: mention it once, briefly — don't turn this into a per-item confirmation loop the way a gating audit would. This is a PR-body note, not a stop condition.

If the subagent fails or times out: note "Plan completion audit unavailable this run." in the PR body and continue.
