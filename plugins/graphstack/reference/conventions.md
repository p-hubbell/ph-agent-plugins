# graphstack shared conventions

Every skill in this plugin reads/writes state under this convention. Load this file (or have it summarized to you) before writing any stage skill so state formats agree across stages.

## State directory

All state lives in the target repo at `_docs/graphstack/` (never a global/home-dir path — this plugin is scoped to one project per invocation):

```
_docs/graphstack/
  brief.md               # Think output: problem framing, target user, core wedge
  plan.md                 # Plan output: scope + architecture decisions
  tasks.md                 # backlog index table: slug | title | status
  backlog/<slug>.md        # one file per task, see task-template.md below
  review-log.jsonl         # append-only ledger, one JSON object per line
  evidence.jsonl           # append-only ledger, one JSON object per line
  learnings.md             # Reflect output, compounding across runs
  sessions/<ts>.md          # context-save/context-restore checkpoints, ts = YYYY-MM-DDTHHMM
```

Create `_docs/graphstack/` and subdirectories on first write if they don't exist. Never delete or truncate `review-log.jsonl`, `evidence.jsonl`, `tasks.md` history, or `learnings.md` — these are append-only/accumulating.

## Task file format

Identical to the existing `graph-engineer` plugin's task template (`plugins/graph-engineer/skills/plan/reference/task-template.md`) — reuse it verbatim, don't reinvent:

```markdown
---
status: open
---

# <Task title>

## Goal

One or two sentences: the outcome this task delivers, not how to build it.

## Acceptance Criteria

- [ ] Checkable statement a QA engineer can verify true/false by inspection or a test run.

## Out of Scope

- Anything adjacent this task explicitly does not cover.

## Constraints

- Technical or product constraints. "None." if empty.
```

Status enum: `open` -> `groomed` -> `in-progress` -> `implemented` -> `reviewed` -> `tested` -> `done`, or `review-failed`/`qa-failed` (loops back to `implemented` for re-work). Only the orchestrator (the skill driving the loop, not an agent) advances status — agents move a task at most one step forward from where they found it, same rule the existing plugin already enforces.

Appended by later stages, in order, never removed once present:
- `## Implementation Notes` — added by the Build stage's software-engineer agent.
- `## Review Log` — added by the Review stage, one entry per pass.
- `## QA Log` — added by the Test stage's qa-engineer agent, one entry per pass.

## review-log.jsonl schema

One line per verdict, from either the Review stage or the Test stage:

```json
{"stage": "review", "task": "<slug>", "commit_sha": "<sha>", "verdict": "PASS", "summary": "...", "ts": "<ISO8601>"}
{"stage": "test", "task": "<slug>", "commit_sha": "<sha>", "verdict": "FAIL", "summary": "...", "ts": "<ISO8601>"}
```

`stage` is always `"review"` or `"test"`. `verdict` is always `"PASS"` or `"FAIL"`. `commit_sha` is `git rev-parse HEAD` at the time the verdict was produced.

## evidence.jsonl schema

Written only by the Test stage, after a PASS verdict:

```json
{"task": "<slug>", "commit_sha": "<sha>", "what_ran": "e.g. `bun test`, or specific manual verification steps", "ts": "<ISO8601>"}
```

## Ship's gate check

Before Ship proceeds for a task, it must find, for the task's current `commit_sha` (`git rev-parse HEAD`):
- A `review-log.jsonl` line with `stage: "review"`, matching `task` and `commit_sha`, `verdict: "PASS"`.
- A `review-log.jsonl` line with `stage: "test"`, matching `task` and `commit_sha`, `verdict: "PASS"`.
- An `evidence.jsonl` line with matching `task` and `commit_sha`.

If any commit_sha in the ledgers is stale (doesn't match current HEAD for files touched by the task) or missing entirely, Ship stops and tells the user exactly what's missing (e.g. "no Test evidence for commit abc123 — run /graphstack:test first") rather than shipping anyway.

## Naming

Skill names in this plugin: `think`, `plan`, `build`, `review`, `test`, `ship`, `land-and-deploy`, `reflect`, `pipeline` — invoked as `/graphstack:<name>`. Do not use `checkpoint` as a skill name (Claude Code reserves it for native session rewind); this plugin's Reflect stage uses `sessions/<ts>.md` + the `reflect` skill name instead.

## Tone/format template

Match the existing `graph-engineer` plugin's file shape exactly:
- `SKILL.md` frontmatter: `name`, `description` (a specific, action-oriented one-liner describing when to invoke it and what it hands off to/from).
- Agent files: frontmatter `name`, `description`, `model` (`sonnet` for analysis/writing roles, `opus` for implementation), `tools` (minimal set actually needed — never grant `Write`/`Edit` to a role that shouldn't touch code or files it doesn't own).
- Prose style: short imperative instructions, numbered process steps, a trailing `## Rules` section with hard constraints. No filler, no marketing language carried over from gstack's docs.
