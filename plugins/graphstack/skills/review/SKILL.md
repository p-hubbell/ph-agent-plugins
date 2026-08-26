---
name: review
description: Use after Build to independently review an implemented task's diff for structural issues Test won't catch — SQL safety, race conditions, LLM trust boundaries, and other checklist/specialist findings. Fourth stage of the graphstack pipeline; run on a task with status `implemented`, before /graphstack:test.
---

# Review: independent code review

You (the orchestrator) drive this stage directly — the checklist scan and the Fix-First triage are your judgment calls, not delegated to an agent. Only the specialist deep-dives (Step 3) run as parallel subagents. See `reference/conventions.md` (this plugin's root) for the state-dir and task-file rules this and every other stage follow.

Review is independent code review: it finds structural problems in the diff — the kind tests don't catch. It is not Test (functional correctness against Acceptance Criteria — that's the next stage), and it does not implement new work. It only applies obvious, mechanical fixes (Step 4) and asks about everything else.

If `_docs/graphstack/tasks.md` doesn't exist yet, tell the user to run `/graphstack:plan` first.

## 1. Read the task and the diff

1. Read the task file at `_docs/graphstack/backlog/<slug>.md`. Confirm its status is `implemented` — if not, stop and tell the user which stage to run instead (`open`/`groomed`/`in-progress` → run `/graphstack:build` first; already `reviewed`/`tested`/`done` → nothing to do here).
2. Read its `## Implementation Notes` section (written by Build) for what changed and why.
3. Get the diff to review:
   - If there are uncommitted changes (`git status --porcelain` is non-empty), the diff is `git diff HEAD` (staged + unstaged).
   - Otherwise, assume Build's work is the tip commit: the diff is `git diff HEAD~1 HEAD`.
4. Note `git rev-parse HEAD` — this is the `commit_sha` you'll log in Step 5. If you diffed uncommitted changes, tell the user they need to commit before `/graphstack:ship` will find a matching commit for this verdict.

If there's no diff at all (nothing uncommitted, and the tip commit doesn't touch the files named in Implementation Notes), stop and report: "Nothing to review — no diff found for this task."

## 2. Two-pass checklist scan

Read `checklist.md` in this skill's directory. Apply it against the diff in two passes:

- **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Shell Injection, Enum & Value Completeness. Highest severity — Fix-First leans toward asking the user on these.
- **Pass 2 (INFORMATIONAL):** the remaining categories in the checklist. Lower severity, still actioned.

If the diff touches frontend files (`.tsx`, `.jsx`, `.vue`, `.css`, templates, and similar), also read `design-checklist.md` in this skill's directory and run it as a third pass. Skip it silently otherwise.

Respect each checklist's suppressions — don't flag what it explicitly says not to. Every finding needs `file:line`, a one-line problem, a one-line fix, and a confidence score (1-10) — only report findings you can point at a specific line and quote the motivating code for; a finding you can't verify that way is speculation, not a finding.

## 3. Specialist dispatch (parallel)

Not every specialist applies to every change. Select based on what the diff actually touches:

- **Always, if the diff is 50+ changed lines:** `testing.md`, `maintainability.md`. Below that threshold, skip specialist dispatch entirely — note "Small diff (N lines) — specialists skipped" and go to Step 4 with just the checklist findings.
- **`security.md`** — if the diff touches auth/session/permission code, or touches backend code and is 100+ lines.
- **`performance.md`** — if the diff touches backend or frontend application code (not just docs/config/tests).
- **`data-migration.md`** — if the diff touches migration files or schema definitions.
- **`api-contract.md`** — if the diff adds or changes an API endpoint, route, or public interface.

Read each selected specialist's file from this skill's `specialists/` directory — its content **is** the brief for that dispatch, not background reading for you.

**Launch every selected specialist in a single message, one Agent tool call per specialist**, so they run in parallel with fresh, unbiased context. This is the pipeline's one real fan-out/merge shape — don't run them sequentially, and don't fold their work into your own pass.

Each specialist's prompt includes:
- The full text of its checklist file.
- The diff, or the exact command to reproduce it (`git diff HEAD` or `git diff HEAD~1 HEAD`, matching what you used in Step 1).
- Its required output: one JSON object per finding, on its own line, matching the schema at the top of its checklist file. `NO FINDINGS` and nothing else if it found nothing. No preamble, no commentary.

Dispatch config: `subagent_type: "general-purpose"`, `run_in_background: false` on every call — all specialists must finish before you can merge and triage. If a specialist fails or times out, note it and continue with whoever succeeded; partial specialist coverage beats none.

Print what you dispatched: "Dispatching N specialists: [names]. Skipped: [names] (scope not detected)."

### Merge specialist findings

1. Parse each specialist's output. Skip `NO FINDINGS` outputs and any line that isn't valid JSON.
2. Compute each finding's fingerprint (its `fingerprint` field if present, else `path:line:category`). Where two or more findings share a fingerprint, keep the highest-confidence one, tag it "MULTI-SPECIALIST CONFIRMED (a + b)", and boost its confidence by +1 (cap 10).
3. Apply confidence gates: 7+ shown normally; 5-6 shown with "medium confidence, verify this is actually an issue"; below 5, drop from the main findings list (note the count, don't itemize each one).
4. Compute the PR Quality Score: `max(0, 10 - (critical_count * 2 + informational_count * 0.5))`, capped at 10. This is a signal for the Review Log, not a gate on the verdict.

### Red Team (conditional)

If the diff is 200+ lines, or any specialist returned a CRITICAL finding, dispatch one more specialist: `red-team.md`. Give it the merged findings from the other specialists (so it knows what's already caught) plus the diff, and tell it explicitly to find what the others missed — not to re-run their checklists. Merge its findings the same way as the others. If it returns `NO FINDINGS`, note "Red Team: no additional issues found."

## 4. Fix-First triage

Every finding gets action — not just critical ones. Combine the checklist findings (Step 2) and the merged specialist findings (Step 3).

1. Classify each finding AUTO-FIX or ASK using the Fix-First Heuristic in `checklist.md`. Critical findings lean ASK; informational findings lean AUTO-FIX. Any finding carrying a `test_stub` is always ASK regardless of category — show the proposed test alongside the fix so the user approves both together.
2. Apply every AUTO-FIX directly. Report each as `[AUTO-FIXED] [file:line] Problem → what you did`.
3. If ASK items remain, batch them into a single AskUserQuestion: one numbered item per finding with its severity, the problem, the recommended fix, and options **A) Fix as recommended / B) Skip**, plus one overall recommendation line. (3 or fewer ASK items may be asked individually instead of batched.)
4. Apply fixes for everything the user approved and report what changed. Anything the user chose to skip stays unresolved — that's what drives the verdict below.

If every finding was AUTO-FIX, skip the question entirely.

If a finding describes real work that's genuinely out of scope for this task — not a mechanical fix, not something this task should absorb — don't force it into Fix-First. Note it as a candidate TODO instead, formatted per `TODOS-format.md` in this skill's directory, and mention it in the Review Log. It doesn't block the verdict.

## 5. Record the verdict

**Verdict:** FAIL if any CRITICAL finding is unresolved after Fix-First (the user chose Skip on it, or it was never offered a fix). PASS otherwise — unresolved informational findings don't block PASS.

1. Append a new entry to the task file's `## Review Log` section (create it if absent — never overwrite or delete prior entries):
   - Timestamp
   - Specialist dispatch summary (who ran, who was skipped and why)
   - Findings by category: what was auto-fixed, what the user approved, what was skipped and whether that's safe to leave or is what's driving a FAIL
   - PR Quality Score
   - One line, verbatim: `VERDICT: PASS` or `VERDICT: FAIL`
2. Append one line to `_docs/graphstack/review-log.jsonl` per the schema in `reference/conventions.md`: `{"stage": "review", "task": "<slug>", "commit_sha": "<sha from Step 1>", "verdict": "PASS"|"FAIL", "summary": "<one line>", "ts": "<ISO8601>"}`.
3. Set the task file's frontmatter `status` to `reviewed` (PASS) or `review-failed` (FAIL).

## 6. Hand off

- **PASS:** tell the user the task is reviewed (clean, or clean with informational findings intentionally left open), and that `/graphstack:test` is next.
- **FAIL:** tell the user which unresolved CRITICAL finding(s) are blocking, and that re-invoking `/graphstack:build` on this task is next — the Review Log entry you just wrote is what the engineer needs to fix; they shouldn't have to re-derive your investigation.

## Rules

- Never fix anything beyond what Step 4 classifies as AUTO-FIX or an approved ASK item — Review verifies and mechanically corrects, it does not implement.
- Never skip Step 3's parallel dispatch by doing the specialists' work yourself inline — if a specialist is in scope, it runs as its own Agent call, in the same batched message as its siblings.
- Never advance status past `reviewed`/`review-failed` — that's as far as this stage goes; Test and Ship own everything after.
- A finding you can't point at a specific `file:line` with the motivating code quoted is not a finding — drop it or mark it unverified, don't report speculation as fact.
