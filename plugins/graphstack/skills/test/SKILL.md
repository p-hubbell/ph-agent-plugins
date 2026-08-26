---
name: test
description: Fifth stage of graphstack. Runs after Review passes (task status `reviewed`) — functional/acceptance verification of a task's Acceptance Criteria via the qa-engineer agent, distinct from Review's code-quality pass. Records the verdict to review-log.jsonl and evidence.jsonl and advances task status. Hands off to /graphstack:ship on PASS or loops back to /graphstack:build on FAIL.
---

# Test: verify Acceptance Criteria

You (the orchestrator) own the ledgers and status transitions here — the
`qa-engineer` agent only investigates and reports. See
`reference/conventions.md` (this plugin's root) for the state-dir layout,
task file format, and the `review-log.jsonl`/`evidence.jsonl` schemas this
stage writes to. Load `reference/issue-taxonomy.md` in this skill directory
if you need the severity/category definitions the agent's findings use.

## Process

1. **Find the task.** If given a task file path, use it. Otherwise read
   `_docs/graphstack/tasks.md` for the next task with status `reviewed`. If
   none exists, tell the user there's nothing to test right now (and, if the
   backlog has tasks stuck earlier in the pipeline, suggest `/graphstack:review`
   first) and stop.
2. **Confirm eligibility.** The task file's frontmatter `status` must be
   `reviewed`. If it's anything else (`open`, `groomed`, `in-progress`,
   `implemented`, `qa-failed`, `tested`, `done`), stop and tell the user why
   this task isn't ready for Test (e.g. "still `implemented` — run
   `/graphstack:review` first").
3. **Invoke `qa-engineer`** with the task file path. It reads the Acceptance
   Criteria, verifies each one (running tests/commands where that's the
   fastest way to confirm behavior), classifies any defects by severity and
   category, appends a `## QA Log` entry to the task file, and ends with
   `VERDICT: PASS` or `VERDICT: FAIL`. It never touches status or the
   ledgers — that's this step's job, next.
4. **Record `commit_sha`** via `git rev-parse HEAD` at the moment the verdict
   was produced.
5. **On PASS:**
   - Append one line to `_docs/graphstack/review-log.jsonl`:
     `{"stage": "test", "task": "<slug>", "commit_sha": "<sha>", "verdict": "PASS", "summary": "<short summary of what was verified>", "ts": "<ISO8601>"}`
   - Append one line to `_docs/graphstack/evidence.jsonl`:
     `{"task": "<slug>", "commit_sha": "<sha>", "what_ran": "<test command output or manual verification steps, taken from the QA Log entry — never invented>", "ts": "<ISO8601>"}`
   - Set the task file's frontmatter `status` to `tested`.
   - Report PASS to the user with the summary, and that `/graphstack:ship` is next.
6. **On FAIL:**
   - Append one line to `_docs/graphstack/review-log.jsonl` with
     `"verdict": "FAIL"` and a summary of what failed (schema as above).
   - Do **not** write to `evidence.jsonl` — evidence is only recorded on PASS.
   - Set the task file's frontmatter `status` to `qa-failed`.
   - Report FAIL to the user and hand back to `/graphstack:build` for
     re-work — the `## QA Log` entry `qa-engineer` appended already has the
     per-criterion detail the engineer needs, don't re-summarize it into a
     separate message.

## Hand-off

- **PASS** → next is `/graphstack:ship`.
- **FAIL** → loop back to `/graphstack:build`, which will re-invoke
  `software-engineer` against the same task file (it reads the fresh
  `## QA Log` entry) and re-run Review before Test runs again.

## Rules

- Only this skill writes to `review-log.jsonl`, `evidence.jsonl`, or a task
  file's `status` field — `qa-engineer` never does either, matching how the
  orchestrator (not the agent) owns status transitions elsewhere in this
  plugin.
- Never invent `what_ran` in the evidence record — pull it directly from what
  `qa-engineer` reports it actually executed in the QA Log entry.
- Never truncate or reorder `review-log.jsonl` or `evidence.jsonl` — both are
  append-only ledgers, oldest entry first.
- Don't run `qa-engineer` against a task that isn't `reviewed` — Test verifies
  acceptance behavior on top of code that already passed a quality review; running
  it earlier just re-derives what Review already covers, out of order.
- If `qa-engineer` fails the same task 3 times in a row, stop looping and
  surface that to the user instead of bouncing it back to Build indefinitely.
