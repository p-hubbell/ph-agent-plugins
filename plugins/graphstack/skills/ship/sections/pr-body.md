# Create or update the PR

## Idempotency check

Check whether a PR already exists for this branch:

```bash
gh pr view --json url,number,state -q 'if .state == "OPEN" then "PR #\(.number): \(.url)" else "NO_PR" end' 2>/dev/null || echo "NO_PR"
```

**If an open PR exists:** update its body with `gh pr edit --body-file <tmpfile>`, regenerated from scratch using this run's fresh results (test coverage note, plan completion note, adversarial note, CHANGELOG entry). Never reuse a stale body from a prior run. Print the existing URL and continue to the parent skill's Step 11.

**If no PR exists:** build the body below, write it to a temp file, and create the PR from that file.

## Body template

```markdown
## Summary
<Summarize every commit being shipped. Run `git log <base>..HEAD --oneline` to enumerate them.
Exclude the version/CHANGELOG bookkeeping commit — that's not a substantive change. Group the
rest into logical sections (e.g. "**New capability**", "**Fixes**", "**Infrastructure**"). Every
substantive commit must appear somewhere in the summary.>

## Tasks shipped
<One line per task in scope, from _docs/graphstack/tasks.md:>
- `<slug>`: <title> — commit `<sha>`

## Verification Evidence
<For each task, cite the gate-check evidence from Step 1 of the parent skill — this is what makes
the ship trustworthy, so don't paraphrase it away:>
- `<slug>` — Review: PASS (commit `<sha>`, <date>) · Test: PASS (commit `<sha>`, <date>) · Evidence: <what_ran, from evidence.jsonl>

## Test Coverage
<The note produced by sections/test-coverage.md, or "Coverage audit unavailable this run." if the
subagent failed.>

## Plan Completion
<The note produced by sections/plan-completion.md, or "No plan.md found — skipped." if there was
nothing to check against.>

## Adversarial Review
<The note produced by sections/adversarial.md, or "Adversarial review unavailable this run." if the
subagent failed. Findings here are informational — the hard gate already passed in Step 1.>

## Test plan
- [x] Review verdict: PASS (see Verification Evidence)
- [x] Test verdict: PASS (see Verification Evidence)
```

## Title

`v<NEW_VERSION> <type>: <summary>` if a version was bumped this run (Step 6 of the parent skill); otherwise just `<type>: <summary>`. Keep it short — the body carries the detail.

## Create

```bash
PR_BODY_FILE=$(mktemp)
cat > "$PR_BODY_FILE" <<'PR_BODY_EOF'
<body from above>
PR_BODY_EOF
gh pr create --base <base> --title "<title>" --body-file "$PR_BODY_FILE"
rm -f "$PR_BODY_FILE"
```

Output the PR URL — this is what the parent skill hands off to the user, and what unlocks task-status updates in its Step 11.

**If `gh` is unavailable or `gh pr create` fails for a reason other than "PR already exists":** print the branch name and remote URL, and tell the user to open the PR manually via the web UI. Do not treat this as fatal to the rest of the skill — the branch is pushed and ready; only the automatic PR creation failed. Do not mark any task `done` in this case (parent skill's Step 11 already covers this).
