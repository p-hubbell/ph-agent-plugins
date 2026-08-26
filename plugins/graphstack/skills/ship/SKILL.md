---
name: ship
description: Use once one or more tasks reach status `tested` and are ready to leave the branch — gate-checks the review/test ledgers for each task, syncs the base branch, bumps the version, updates the CHANGELOG, writes a PR body, commits, pushes, and opens a PR via `gh`. Sixth stage of the graphstack pipeline; run after /graphstack:test, before /graphstack:land-and-deploy or /graphstack:reflect. Does NOT merge — a human merges the PR, or invokes /graphstack:land-and-deploy explicitly.
---

# Ship: gate-check, then open the PR

You are the orchestrator for this stage — drive it directly, don't hand the whole thing to an agent. Read `reference/conventions.md` (this plugin's root) before starting if you haven't already this session; the "Ship's gate check" section there is the exact logic Step 1 below implements, and the "State directory" section defines every file path used here.

Ship's job ends when the PR is open. It never merges, never deploys, and never touches `main` (or whatever the base branch is). Merging is a human action, or an explicit `/graphstack:land-and-deploy` invocation later in this same session.

## Step 1: Gate check (hard stop on failure)

This runs before anything else, every time — including re-runs. Never skip it because a prior `/graphstack:ship` run already passed it once; the ledgers may have changed since.

1. Read `_docs/graphstack/tasks.md`. Collect every task with status `tested`. If the user named a specific task (slug or title), scope to just that one — it must be `tested`, otherwise stop: "`<slug>` is `<status>`, not `tested` — nothing to gate-check." If no tasks are `tested` and none was named, stop: "No tasks are ready to ship. Run `/graphstack:test` on a `reviewed` task first."

2. For each task in scope, read `_docs/graphstack/review-log.jsonl` and `_docs/graphstack/evidence.jsonl` (append-only JSONL — one JSON object per line) and find:
   - The most recent line with `stage:"review"`, matching `task:"<slug>"`.
   - The most recent line with `stage:"test"`, matching `task:"<slug>"`.
   - The most recent line in `evidence.jsonl` matching `task:"<slug>"`.

3. Verify, in order, and stop at the first failure with the exact reason (don't bundle a batch of vague failures — name the specific task and specific gap):
   - **No review entry found** → stop: "no Review verdict for `<slug>` — run `/graphstack:review` first."
   - **Latest review verdict is FAIL** → stop: "latest Review verdict for `<slug>` is FAIL (commit `<sha>`) — fix the findings and re-run `/graphstack:review`."
   - **No test entry found** → stop: "no Test verdict for `<slug>` — run `/graphstack:test` first."
   - **Latest test verdict is FAIL** → stop: "latest Test verdict for `<slug>` is FAIL (commit `<sha>`) — fix and re-run `/graphstack:test`."
   - **No evidence entry found** → stop: "no Test evidence for commit `<sha>` — run `/graphstack:test` first." (this is the exact phrasing conventions.md specifies)
   - **The three `commit_sha` values (review PASS, test PASS, evidence) don't all match** → stop: "`<slug>` was reviewed at `<sha-a>` but tested at `<sha-b>` — re-run `/graphstack:review` and `/graphstack:test` against the same commit."
   - **The agreed-upon `commit_sha` is not an ancestor of current `HEAD`** (`git merge-base --is-ancestor <sha> HEAD`, non-zero exit means stale) → stop: "Review/Test evidence for `<slug>` is recorded at commit `<sha>`, which isn't in this branch's history — was it rebased or amended away? Re-run `/graphstack:review` and `/graphstack:test`."

4. If every task in scope clears all five checks, print a one-line confirmation per task (`<slug>: Review PASS, Test PASS, evidence @ <sha-short>`) and continue to Step 2. Nothing else in this skill runs until every task in scope is clean.

## Step 2: Detect platform and base branch

```bash
git remote get-url origin 2>/dev/null
command -v gh >/dev/null 2>&1 && gh auth status 2>/dev/null
```

This plugin ships through GitHub via the `gh` CLI. If `gh` isn't on PATH, or `gh auth status` fails, stop: "`gh` CLI isn't available or isn't authenticated — install/auth it, or open the PR manually once I've pushed the branch." (Still complete Steps 3-10 up to the push; only the PR-creation step is blocked.)

Determine the base branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`. If that fails, fall back to `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'`, then `main`, then `master`. Use this branch everywhere below as `<base>`.

## Step 3: Preflight

1. Current branch: `git branch --show-current`. If it equals `<base>`, stop: "You're on `<base>`. Ship from a feature branch."
2. `git status` (never `-uall`) — note any uncommitted changes; they'll be included in Step 8's commit.
3. `git diff <base>...HEAD --stat` and `git log <base>..HEAD --oneline` — this is what's being shipped. Skim it before writing the CHANGELOG/PR body later.

## Step 4: Sync the base branch

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

If there are merge conflicts, try to auto-resolve only the obviously-mechanical ones (VERSION, CHANGELOG ordering). If anything else conflicts, stop and show the conflicting files — do not guess at a resolution for application code.

## Step 5: Audit and compose ship content

These three sections are informational, not gating — the hard gate already happened in Step 1. Their job is to produce good content for the PR body and to surface anything worth a human's attention before it ships, not to re-litigate a verdict the Review/Test stages already reached.

Read each section in full when you reach it — do not work from memory or summary:

- Read `sections/test-coverage.md` and follow it to produce a coverage note for the PR body.
- Read `sections/plan-completion.md` and follow it to produce a plan-completion note for the PR body.
- Read `sections/adversarial.md` and follow it to produce an adversarial-review note for the PR body.

Dispatch each as a subagent (`general-purpose`) per that section's own instructions — parallel is fine, they're independent reads of the same diff. If a subagent fails or times out, note "audit unavailable" in that PR body section and continue; none of these three block shipping.

## Step 6: Version bump

Look for a `VERSION` file at the repo root (plain semver, e.g. `1.4.2`). If absent, check `package.json`'s `"version"` field. If neither exists, skip this step entirely and note in the PR body: "No version file found — skipped version bump." Don't invent a versioning scheme a project doesn't already have.

If a version source exists:

1. Classify the bump level from the diff: **PATCH** for fixes/small changes, **MINOR** for new capability with no breaking change, **MAJOR** for a breaking change. Default to PATCH unless the diff clearly adds a new feature/route/module (→ MINOR) or breaks an existing contract (→ MAJOR).
2. **PATCH bumps auto-proceed.** For **MINOR or MAJOR**, ask the user to confirm the level via AskUserQuestion before writing it — this is a real decision, not a formality.
3. Write the new version to `VERSION` and/or `package.json` (whichever exists; update both if both do, keeping them in sync).

## Step 7: CHANGELOG

Read `sections/changelog.md` and follow it in full.

## Step 8: Commit

Stage and commit everything from Steps 5-7 plus any uncommitted work noted in Step 3. Prefer a small number of logically-scoped commits over one giant commit (e.g. one commit for the version/CHANGELOG bump, separate from any leftover uncommitted work) — but don't over-engineer this into gstack's full bisectable-commit machinery. A single commit is fine when the leftover diff is small. Never `git add -A`; stage intentional files only.

Commit message format: `<type>: <summary>` first line (type = feat/fix/chore/refactor/docs), optionally a body. The version/CHANGELOG commit specifically:

```
chore: bump version and changelog (vX.Y.Z)
```

## Step 9: Push

```bash
git push -u origin $(git branch --show-current)
```

If the branch is already up to date with its remote, skip the push silently and continue.

## Step 10: Open or update the PR

Read `sections/pr-body.md` and follow it in full — it covers the idempotency check (PR already open → update instead of create), the body template, and the `gh pr create`/`gh pr edit` invocations.

## Step 11: Mark tasks done

Only after the PR is confirmed open (Step 10 produced a URL): for every task in scope, set its frontmatter `status` to `done` and update its row in `_docs/graphstack/tasks.md`. If PR creation failed (Step 2's `gh` warning, or an error partway through Step 10), leave every task's status at `tested` — nothing shipped yet, nothing should look shipped.

## Step 12: Hand off

Tell the user:
- The PR URL.
- Which tasks shipped (slug + title, one line each).
- That `/graphstack:land-and-deploy` is available now if they want to merge, deploy, and verify in this same session — otherwise a human merges the PR on their own schedule, and `/graphstack:reflect` is the next stage once it's merged.

## Rules

- Never skip Step 1's gate check, on this run or any re-run — it is the one thing standing between "the ledgers say this is safe" and "it actually is."
- Never merge the PR, never push to `<base>` directly, never auto-approve. This stage's job ends at "PR opened."
- Never mark a task `done` before the PR actually exists.
- Never invent a VERSION/CHANGELOG convention the repo doesn't already have — skip and note, don't impose.
- Report what happened at each step as you go (gate check result, sync result, version bump, PR URL) — don't run silently and summarize once at the end.
