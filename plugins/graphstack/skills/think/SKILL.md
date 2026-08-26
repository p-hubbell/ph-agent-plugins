---
name: think
description: Use when the user has flexible input for something to build — a vague idea, a one-liner, a pasted spec, a markdown file, or a folder of docs — and wants it turned into a clear problem brief before planning starts. First stage of the graphstack pipeline; run before /graphstack:plan.
---

# Think: idea -> brief

Turns whatever the user gave you into `_docs/graphstack/brief.md`: the problem framing, the target user, and the narrowest useful version worth building. See `reference/conventions.md` (this plugin's root) for the state-dir rules this and every other stage follow.

## 1. Load prior learnings

If `_docs/graphstack/learnings.md` exists, read it before doing anything else. It's written by the Reflect stage from prior runs — factor any relevant lessons (recurring mistakes, standing preferences, things that turned out to matter) into how you run discovery and what you write into the brief. If it doesn't exist yet, skip this step; there's nothing to load.

## 2. Judge the shape of the input

- **Vague** (a one-liner, a rough idea, "something like X but for Y", no real detail to go on): delegate to the `facilitator` agent. It runs forcing-questions discovery — one question at a time — until the idea has a clear enough shape to write a checkable brief.
- **Already detailed** (the user pasted a spec, handed you a markdown file, or pointed at a folder of docs): don't re-interrogate from scratch. Read what's given directly, then confirm scope and any real ambiguities with the user — batched questions are fine here, this isn't blank-slate discovery. Use AskUserQuestion or plain prose, whichever fits the number of open items.

Either way, you're aiming for the same three things before writing the brief: the problem being solved, who it's for, and the smallest version of it worth shipping.

## 3. Write the brief

Create `_docs/graphstack/` if it doesn't exist yet. Write `_docs/graphstack/brief.md`:

```markdown
# Brief: <title>

## Problem
What's broken or missing today, stated plainly. Who feels it and how.

## Target User
The specific person or role this is for. Not a category — a user.

## Core Wedge
The narrowest version of this that's still worth building: what it does,
what it deliberately leaves out for now.

## Assumptions
Anything taken as given that the user should flag if wrong.

## Open Questions
Anything still unresolved that Plan or Build will need to settle.
```

Omit `## Open Questions` if there genuinely aren't any — don't pad it. Keep the whole thing tight: this is a brief, not a spec. Plan is where it gets decomposed into tasks.

## 4. Hand off

Tell the user:
- Where the brief was written (`_docs/graphstack/brief.md`).
- A one-line summary of the wedge you landed on.
- That `/graphstack:plan` is next, and it will turn this brief into a task backlog.

## Rules

- Never write or edit application code in this stage — Think only produces the brief.
- Don't skip discovery just because the user seems impatient. If they push back, cut to the two most load-bearing questions (per the `facilitator` agent's own escape hatch), but don't skip straight to writing the brief on a genuinely vague idea.
- Don't re-interrogate a user who already gave you a detailed spec — that's the plan skill's judgment call too, and re-litigating it here wastes their time and erodes trust in the pipeline.
