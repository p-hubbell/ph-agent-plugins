---
name: reflect
description: Seventh and final stage of graphstack, run after Ship (or at the end of a work session) — analyzes what happened this pipeline run and writes durable learnings that feed back into the next Think invocation, closing the loop.
---

# Reflect: close the loop

Analyzes this run's git activity and review/test ledgers, checkpoints the session so it can be resumed cold, and appends compounding lessons to `_docs/graphstack/learnings.md` for the next `/graphstack:think` to read. See `reference/conventions.md` (this plugin's root) for the state-dir rules this and every other stage follow.

**Ownership split:** the `retro-analyst` agent (invoked in Step 1) only analyzes and reports back in prose — it never writes any file. This skill (the orchestrator) is the sole writer of `sessions/<ts>.md` (Step 2) and `learnings.md` (Step 3). If you're ever unsure who writes what, the rule is: the agent computes, the skill persists.

## 1. Run the retro analysis

Invoke the `retro-analyst` agent. It determines its own analysis window (most recent session checkpoint, else branch divergence from default, else last 7 days — see the agent file), runs the git-log pipeline, and reads `_docs/graphstack/review-log.jsonl` and `_docs/graphstack/evidence.jsonl` for this run's history. It hands back a structured report: commit counts, shipping streak, test-health trend, review/test pass-fail counts, and anything notable. Wait for this before continuing — Steps 2 and 3 both draw on it.

## 2. Write the session checkpoint

Gather current state directly (don't delegate this — it's cheap and this skill owns the file):

```bash
git branch --show-current
git rev-parse HEAD
git status --short
```

Read `_docs/graphstack/tasks.md` if it exists and list every row whose status is not `done` — that's the remaining backlog.

Create `_docs/graphstack/sessions/` if it doesn't exist. Write `_docs/graphstack/sessions/<ts>.md` where `<ts>` is the current time formatted `YYYY-MM-DDTHHMM` (per conventions.md):

```markdown
---
branch: <current branch>
head_sha: <git rev-parse HEAD>
ts: <YYYY-MM-DDTHHMM>
---

# Session checkpoint: <ts>

## Git state
Branch `<branch>` at `<sha>`. <clean, or a one-line summary of what git status --short shows>

## Decisions made this session
<Bulleted list of real decisions — architecture calls, scope calls, things chosen and why. Pull from conversation history and, if relevant, from plan.md/backlog task notes. If nothing decision-worthy happened this session, say so plainly rather than padding.>

## Retro summary
<Condensed version of the retro-analyst report from Step 1 — commit count, shipping streak, test-health trend, review/test pass-fail. A few lines, not the full report.>

## Remaining backlog
<Table or list of tasks.md rows not yet `done`, with their current status. "Backlog empty — all tasks done." if none.>

## Notes
<Anything a cold read of this file would need: blockers, open questions, things tried that didn't work.>
```

This file is the context-save/context-restore pattern: written once per Reflect run, never overwritten, so a future `/graphstack:reflect` or a manual read of the latest file is enough to resume cold. Never delete or truncate prior checkpoints.

## 3. Append to learnings.md

Create `_docs/graphstack/learnings.md` if it doesn't exist. This file is append-only and durable — it compounds across every graphstack run in this project, and Think reads it at the start of every future invocation. Append (don't rewrite existing entries):

```markdown
## <YYYY-MM-DD> — <one-line summary of this run>

- **Worked:** <a pattern, approach, or decision that paid off this run — specific enough to repeat deliberately next time>
- **Didn't work:** <a pattern, approach, or assumption that cost time or caused rework — specific enough to avoid next time>
- **Watch for:** <anything from the retro-analyst report worth flagging forward — a recurring fix-chain, a test-health dip, a file that keeps churning>
```

Keep entries terse — a few bullets, not a report. Only log genuine, durable discoveries: something a future Think/Plan/Build pass would actually benefit from knowing. Skip an entry (or a bullet within it) if this run didn't produce anything worth carrying forward — don't manufacture lessons to fill the template. Never remove or edit a prior entry; if a later run contradicts an earlier lesson, add a new entry noting the correction rather than rewriting history.

## 4. Hand off

Tell the user, briefly:
- What shipped this run (from the retro summary — commit count, tasks completed).
- What's left in the backlog (from Step 2), if anything.
- Where the checkpoint and learnings files are (`_docs/graphstack/sessions/<ts>.md`, `_docs/graphstack/learnings.md`).
- That the next `/graphstack:think` will read `learnings.md` automatically.

## Rules

- Never write or edit application code or task files in this stage — Reflect only produces the checkpoint and the learnings entry.
- `retro-analyst` is read-only. If it ever proposes writing a file, decline and do the write yourself from its report.
- Never truncate or overwrite `sessions/*.md`, `learnings.md`, `review-log.jsonl`, or `evidence.jsonl` — all are append-only per conventions.md.
- Single-project scope only. gstack's `/retro global` (cross-project retrospective) is a plausible future extension for this plugin but is explicitly out of scope for v1 — don't attempt to analyze other projects or a global state directory.
- If `review-log.jsonl` and `evidence.jsonl` don't exist yet (Reflect run before any Build/Review/Test cycle), that's not an error — note it in the checkpoint and learnings entry and continue with whatever git history exists.
