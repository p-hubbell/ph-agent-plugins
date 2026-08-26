---
name: pipeline
description: Runs the full graphstack pipeline end to end — Think -> Plan -> Build -> Review -> Test -> Ship -> Reflect — from a single invocation. Use when the user hands over an idea, a spec file/folder, or just says "build this" and wants the whole loop driven for them, rather than running one stage at a time. Each stage is also independently invocable (/graphstack:think, /graphstack:plan, etc.) if the user wants to run or re-run just one.
---

# Pipeline: idea to shipped and reflected

You are the meta-orchestrator. You don't reimplement any stage's logic here — you read each stage's own `SKILL.md` when you reach it and execute its instructions in full, in order, the same way gstack's `autoplan` sequences its own review skills by reading them off disk rather than duplicating their content. This file's job is dispatch and sequencing, not stage logic.

See `reference/conventions.md` (this plugin's root) for the state directory layout, task format, and ledger schemas every stage below assumes.

## 1. Figure out where to start

Don't always start at Think — resume wherever the project's state already is:

1. Check `_docs/graphstack/tasks.md`. If it exists and has any task not `done`, this project already has a backlog in flight. Read it, report the current state (how many tasks at each status), and start at whichever stage the earliest non-`done` task needs next (`open`/`groomed` → Build after grooming, `implemented` → Review, `reviewed` → Test, `tested` → Ship, `review-failed`/`qa-failed` → Build). Confirm this with the user in one line before proceeding ("Resuming — 3 tasks in flight, starting at Review for `<slug>`.") rather than silently jumping in.
2. If there's no backlog yet, judge the shape of what the user gave you:
   - **A raw idea, a one-liner, or "build X"** — start at Think.
   - **A markdown file, a spec, or a folder of docs that already reads like a finished plan** (has clear scope, maybe already broken into pieces) — ask the user whether to run it through Think anyway (useful if they want the forcing-questions pressure-test) or skip straight to Plan with it as direct input. Don't assume; this is a genuine judgment call worth one question.
   - **Nothing given at all, just "run the pipeline"** — ask what they want built. Don't guess at a starting idea.

## 2. Run the stages in order

For each stage below, in sequence: read that stage's `SKILL.md` (path given) and execute its instructions completely, including its own hand-off. Do not skip a stage's internal steps because you've "already got the gist" from earlier in this file — the actual process detail lives in the stage file, not here.

1. **Think** — `skills/think/SKILL.md`. Produces `_docs/graphstack/brief.md`.
2. **Plan** — `skills/plan/SKILL.md`. Produces `_docs/graphstack/plan.md` and the task backlog.
3. **Build** — `skills/build/SKILL.md`. Works the backlog to `implemented`, one task at a time, looping internally per its own stop conditions.
4. **Review** — `skills/review/SKILL.md`. Run once per task that Build just brought to `implemented`. On FAIL, that task loops back to Build's `software-engineer` (re-invoke Build for just that task) and back through Review before continuing — don't advance other tasks past it silently, but don't block unrelated tasks either; work the backlog task-by-task through Review→Test→(loop back to Build if needed) before moving to the next task, rather than running all of Review across the whole backlog first.
5. **Test** — `skills/test/SKILL.md`. Run once per task Review just passed. On FAIL, loops back to Build (which re-runs Review after rework, per Test's own hand-off).
6. **Ship** — `skills/ship/SKILL.md`. Once one or more tasks reach `tested`, gate-check and ship them. Ship's own gate check is the real safety net here — trust it, don't re-verify its logic in this file.
7. **Reflect** — `skills/reflect/SKILL.md`. Run once, after Ship, regardless of how many tasks shipped this pass (even zero — Reflect still has value in capturing what was learned and checkpointing state).

`skills/land-and-deploy/SKILL.md` is deliberately **not** in this default sequence — per its own description, it only runs on explicit invocation ("merge it," "deploy it," `/graphstack:land-and-deploy`). If the user asks for that mid-pipeline-run, run it between Ship and Reflect; otherwise Reflect runs directly after Ship with the PR still open.

## 3. Per-task looping through Build→Review→Test

Within one pipeline run, drive each task through Build→Review→Test as a unit before starting the next task, rather than batching all tasks through each stage in turn:

- This keeps failures contained: a task stuck bouncing between Review and Build doesn't block a sibling task that's already through to Test.
- It also matches how the individual stage skills already expect to be invoked — each one operates on "the next eligible task," and eligibility is per-task, not per-batch.

If the user's stop condition only wants the first task done (e.g. "just get the first one shipped"), stop after that task clears Test — don't continue to Ship/Reflect unless they ask, and say plainly that you stopped early per their instruction.

## 4. Stop conditions

Surface these instead of continuing silently, same as every individual stage already does:

- A task fails Review or Test 3 times in a row — stop on that task, report what's failing, ask the user how to proceed (skip it and continue with the rest of the backlog, or hold everything until it's resolved).
- Ship's gate check fails for a reason that isn't self-resolving within this run (e.g. `gh` not authenticated) — report it and stop; don't silently skip Ship and jump to Reflect as if shipping happened.
- The user gave an explicit scope ("just the first MVP," "stop after Build, I want to review the code myself before it goes further") — honor it over running the full seven stages.
- Any individual stage's own stop conditions (each `SKILL.md` has its own) — inherit and respect them; this file doesn't override a stage's judgment about when to halt.

## 5. Report as you go

After every stage completes, tell the user what happened and what's next — the same per-stage hand-off each `SKILL.md` already produces is enough; don't suppress it in service of a single end-of-run summary. A pipeline run through all seven stages is a long operation; silence in the middle reads as broken, not efficient.

## Rules

- Never reimplement a stage's logic inline — always read and follow that stage's actual `SKILL.md` when you reach it. This file is a sequencer, not a summary standing in for the real instructions.
- Never advance a task past what its current stage's `SKILL.md` says to do — in particular, never invoke Ship on a task that isn't `tested`, and never invoke `land-and-deploy` unless the user explicitly asked for it this run.
- Never skip Reflect at the end of a run, even a short one or one that stopped early — it's what makes the next `/graphstack:pipeline` or `/graphstack:think` invocation smarter than this one.
