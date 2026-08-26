---
name: land-and-deploy
description: Optional stage after Ship — merges the open PR, waits for CI, waits for the deploy, runs a lightweight post-deploy check, and offers a revert if something looks broken. Human-checkpoint-heavy by design (merging and deploying are semi-irreversible). Invoke explicitly after /graphstack:ship when the user wants to merge, deploy, and verify in the same session; otherwise a human merges the PR on their own schedule and /graphstack:reflect is the next stage once it lands.
---

# Land and deploy: merge, wait, verify

You are picking up exactly where `/graphstack:ship` left off: the PR is open, nothing is merged. Everything from here on is harder to undo than anything in the pipeline so far — a merge needs a revert commit to undo, and a bad deploy is live in front of users until it's fixed or rolled back. Treat every irreversible step as a step that needs the human to actually say go, not a step you infer consent for from "the user typed the slash command."

This skill is **not** part of the default pipeline. It only runs when explicitly invoked — either by name or because the user said something like "merge it," "deploy it," "land the PR."

## Step 1: Find the PR

1. Confirm `gh` is available and authenticated: `gh auth status`. If not, stop: "`gh` isn't authenticated — run `gh auth login`, then try again."
2. If the user gave a PR number (`#123`) or URL, use it. Otherwise detect from the current branch:
   ```bash
   gh pr view --json number,state,title,url,mergeStateStatus,mergeable,baseRefName,headRefName
   ```
3. Tell the user what you found: "Found PR #NNN — '<title>' (`<head>` → `<base>`)."
4. Validate state:
   - `MERGED` → stop: "This PR is already merged — nothing to land. If you want to check the deploy, look at Step 6 below manually or check your platform's dashboard."
   - `CLOSED` → stop: "This PR was closed without merging. Reopen it first."
   - No PR found → stop: "No open PR for this branch. Run `/graphstack:ship` first."
   - `OPEN` → continue.

## Step 2: Pre-merge checks

```bash
gh pr checks --json name,state,status,conclusion
gh pr view --json mergeable -q .mergeable
```

- Any required check **FAILING** → stop: "CI is failing on this PR: `<list>`. Fix before I merge — I won't land code that hasn't passed CI."
- `mergeable` is `CONFLICTING` → stop: "This PR has merge conflicts with `<base>`. Resolve and push, then try again."
- Checks **PENDING** → continue to Step 3.
- Checks all passed → skip Step 3, go to Step 4.

## Step 3: Wait for CI

```bash
gh pr checks --watch --fail-fast
```

Timeout at 15 minutes. On timeout, stop: "CI has been running 15+ minutes — that's unusual. Check the Actions tab; something may be stuck." On failure, stop and show what broke. On success, continue.

## Step 4: Pre-merge readiness gate (human checkpoint)

This is the last stop before an action that needs a revert commit to undo. Re-verify the same ledgers Ship gated on — time may have passed since the PR opened, and the branch may have moved.

1. Re-run the gate check from `plugins/graphstack/skills/ship/SKILL.md` Step 1, against the tasks this PR covers (read `_docs/graphstack/tasks.md` for tasks at status `done` whose commit history matches this branch, or ask the user which tasks this PR ships if that's ambiguous). Confirm each still has Review PASS + Test PASS + evidence, all at a `commit_sha` that's an ancestor of the PR's current head — a force-push or an amended commit since Ship ran would invalidate this.
2. Confirm CI is green (from Steps 2-3).
3. Build a short readiness summary:
   ```
   PR #NNN — <title> (<head> → <base>)
   Tasks: <slug> (Review PASS, Test PASS, evidence @ <sha>), ...
   CI: PASSED
   ```
4. **Stop and ask the user to confirm** (AskUserQuestion or a plain explicit question if AskUserQuestion isn't available) before merging: "Ready to merge PR #NNN into `<base>`. This is a real merge — undoing it later means a revert commit, not a quiet rollback. Go ahead?" Do not proceed on an ambiguous or implied yes ("ok", "sure") without the user clearly confirming this specific action. If the gate check in step 1 found anything stale or missing, say so plainly and let that inform the ask — don't hide a gap behind a generic "ready?" question.

Only continue to Step 5 once the user explicitly confirms.

## Step 5: Merge the PR

Try auto-merge first (respects repo settings and merge queues):

```bash
gh pr merge --squash --auto --delete-branch
```

If that fails (auto-merge disabled, or nothing left to wait on — both fall through the same way, don't misreport either as "auto-merge is disabled"):

```bash
gh pr merge --squash --delete-branch
```

**After any non-zero exit from `gh pr merge`, never retry the command.** Query authoritative state instead:

```bash
gh pr view --json state,mergeCommit,mergedAt
```

- `state == "MERGED"` → the merge went through (possibly a concurrent merge, or cleanup failed after a successful server-side merge). Say "PR is merged on GitHub" (not "the merge succeeded," in case someone else's merge landed first). Capture `mergeCommit.oid` as the merge SHA.
- `state == "OPEN"` with `autoMergeRequest` non-null → it's queued in a merge queue. Poll `gh pr view --json state -q .state` every 30s, up to 30 minutes, printing progress every 2 minutes ("Still in the merge queue... (<N>m)"). If it flips to `MERGED`, capture the SHA and continue. If it drops back to `OPEN` outside a queue, or times out, stop and report.
- `state == "OPEN"` with no auto-merge pending → genuine failure. Show the `gh` error and the PR state, then stop.
- Permission error → stop: "I don't have permission to merge this PR — a maintainer needs to do it, or check branch protection rules."

Record the merge SHA and merge path (auto/direct/queue) — you'll need the SHA for Step 9 if a revert is ever needed.

## Step 6: Deploy configuration

Look for a documented deploy convention before assuming anything. Check, in order:

1. **`CLAUDE.md`**, a `## Deploy Configuration` section, e.g.:
   ```markdown
   ## Deploy Configuration
   Platform: vercel
   Production URL: https://example.com
   Health check command: npm run smoke-test -- --url https://example.com
   ```
   Read `Production URL:` and `Health check command:` (or `Health check URL:`) if present.
2. **Environment variables** `GRAPHSTACK_HEALTH_CHECK_URL` and `GRAPHSTACK_HEALTH_CHECK_CMD` — a lighter-weight alternative to editing CLAUDE.md.
3. **Platform config files** as a hint only (not a health check by themselves): `vercel.json`/`.vercel`, `netlify.toml`, `fly.toml`, `render.yaml`, `Procfile`, `railway.json`/`.toml`.
4. **A deploy workflow**: `gh run list --branch <base> --limit 5 --json name,status,conclusion,workflowName,headSha`, looking for a name containing `deploy`, `release`, `production`, or `cd`.

Classify what you found:
- **Health check configured** (`GRAPHSTACK_HEALTH_CHECK_CMD`/`URL`, or CLAUDE.md's fields) → Step 8 will run it.
- **No health check configured, but a deploy workflow exists** → Step 7 waits for it; Step 8 runs CI/deploy-status verification only and says plainly that no deeper check ran.
- **Neither** → skip Step 7 (nothing to wait on), and Step 8 reports "no deploy detection configured — merged only, verify manually." This is not a failure; plenty of repos (libraries, CLIs, docs-only changes) have nothing to deploy.

If nothing is configured, mention once: "No health-check URL/command is configured — I can only confirm the merge and CI status, not that production is actually healthy. Add a `## Deploy Configuration` section to CLAUDE.md (or set `GRAPHSTACK_HEALTH_CHECK_URL`/`GRAPHSTACK_HEALTH_CHECK_CMD`) if you want a real post-deploy check next time."

## Step 7: Wait for deploy (only if a deploy workflow was detected)

Match the run to the merge commit SHA from Step 5:

```bash
gh run list --branch <base> --limit 10 --json databaseId,headSha,status,conclusion,workflowName
```

Poll `gh run view <run-id> --json status,conclusion` every 30 seconds. Print progress every 2 minutes ("Deploy still running... (<N>m)"). Timeout at 20 minutes — on timeout, tell the user and ask whether to keep waiting or move on to Step 8 anyway.

If no matching workflow run appears within a couple minutes of merge, and the platform is a known auto-deploy-on-merge platform (Vercel/Netlify), wait 60 seconds for propagation and move to Step 8 without a workflow to poll — this is expected, not a failure.

If the deploy run's `conclusion` is `failure`: **stop and ask** (don't auto-decide) whether to investigate the logs, revert now (Step 9), or continue to the health check anyway in case the failure was in an unrelated step and the site is actually fine.

## Step 8: Post-deploy check

Run exactly one of these, based on Step 6's classification:

**A — Health check configured:** run the configured command or hit the configured URL.
```bash
# command form
eval "$HEALTH_CHECK_CMD"
# URL form
curl -sf -o /dev/null -w '%{http_code}' "$HEALTH_CHECK_URL"
```
Non-2xx status, or a non-zero command exit, is a failure. Report exactly what ran and what it returned.

**B — No health check, but deploy/CI status is known:** report CI status (Step 2/3) and deploy run status (Step 7) as the only verification performed, explicitly: "No deep health check ran — I don't have a URL or command configured. CI passed and the deploy workflow reports `<conclusion>`. That's all I can confirm; check the site yourself if you want more confidence."

**C — Nothing to check:** report "Merged. No deploy or health check detected for this repo — nothing further to verify."

Classify the outcome as **HEALTHY**, **DEGRADED** (check failed or returned something unexpected), or **UNVERIFIED** (case B/C).

## Step 9: Revert (only if something looks broken, and only with explicit confirmation)

Offer this whenever Step 7 reports a deploy failure or Step 8 reports DEGRADED. Never revert automatically — always ask first, and always say plainly what a revert does.

Ask (AskUserQuestion or an explicit question): "The <deploy failed / health check failed: `<specifics>`>. I can revert the merge — that creates a new commit undoing everything from this PR, and (if your deploy is automatic) should roll production back once it lands. Want me to?"

If yes:
```bash
git fetch origin <base>
git checkout <base>
git revert <merge-sha> --no-edit
git push origin <base>
```

- Conflict on revert → stop: "The revert has conflicts — that can happen if something else landed on `<base>` after this merge. The merge commit is `<sha>`; you'll need to resolve `git revert <sha>` by hand."
- Push rejected by branch protection → create a revert PR instead: `gh pr create --title 'revert: <original title>'` and tell the user to merge that one to complete the rollback.
- Success → tell the user plainly: "Reverted and pushed to `<base>`. If your deploy is automatic, it should roll back once that lands — keep an eye on it. The original PR's branch is gone (it was deleted on merge); the tasks it shipped are still marked `done` in `tasks.md` — you'll want to reopen or re-triage them once you've sorted out what broke." Do not touch task status yourself here — that's a decision for the user, not an automatic side effect of a revert.

If no: "Understood — leaving it as is. You can revert manually later with `git revert <merge-sha>`."

## Step 10: Report

Print a short summary:

```
PR #NNN merged into <base> (commit <sha>)
CI: PASSED
Deploy: <workflow conclusion, or "no workflow detected">
Post-deploy check: <HEALTHY / DEGRADED / UNVERIFIED — with the one-line reason>
<REVERTED, if applicable>
```

## Step 11: Hand off

If HEALTHY (or UNVERIFIED with nothing configured — that's the expected steady state for a repo with no deploy target): "Merged and deployed. `/graphstack:reflect` is next once you're ready to capture what was learned this run."

If DEGRADED and not reverted: "Merged, but the post-deploy check found `<issue>` and you chose not to revert. Worth checking manually before calling this done."

If REVERTED: "Reverted — `<base>` is back to its pre-merge state. Fix the issue and run `/graphstack:ship` again when ready."

## Rules

- **Never merge without the Step 4 human confirmation**, even on a re-run, even if the PR looked ready last time you checked — state can change between Ship and Land.
- **Never retry `gh pr merge`** after a non-zero exit; always re-query PR state instead (avoids double-merge attempts and misleading error reporting on races).
- **Never force-push, never `git reset --hard` on a shared branch.**
- **Never auto-revert.** Every revert requires an explicit yes from the user, stated as this specific action — not inferred from "the deploy failed so obviously they'd want a revert."
- **Never invent a health check.** If nothing is configured, say so and report exactly what was actually verified (CI + deploy status) — do not imply a deeper check happened.
- **Narrate as you go.** Merging and deploying are stressful even when they go fine; the user should know what's happening at each step, not just get a final report.
