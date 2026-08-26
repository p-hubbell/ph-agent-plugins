# Test coverage audit

This is informational, not gating — the Test stage already produced a PASS verdict and evidence for the tasks in scope (Step 1 of the parent skill verified this). The job here is to produce an honest coverage snapshot for the PR body, and to surface any gap worth a human's attention. It never blocks shipping.

**Dispatch as a subagent** (`general-purpose`) so this runs in a fresh context window. Give it:

> Audit test coverage for the diff between `<base>` and `HEAD` in this repo. Run `git diff <base>...HEAD` as needed. Do not commit, push, or modify anything — report only.
>
> 1. **Trace every changed code path.** For each changed file, read the full file (not just the hunk). Follow each entry point (route handler, exported function, component) through every branch — conditionals, error handlers, calls into other changed functions.
> 2. **Map user-facing flows and edge cases** for anything user-facing in the diff: the happy path, what happens on invalid input, on a slow/failed dependency, on an empty/boundary state.
> 3. **Check each path against existing tests.** For each branch and each flow, look for a test that exercises it. Score:
>    - ★★★ tests behavior + edge cases + error paths
>    - ★★ happy path only
>    - ★ smoke test / existence check only
> 4. **Output an ASCII diagram** listing each path/flow as `[TESTED]` (with location) or `[GAP]`, plus a summary line: `COVERAGE: N/M paths tested (X%)`.
> 5. **Do not generate new tests here** — this is a read-only audit for a PR body, not a test-writing pass. If a gap looks serious (an error path with no coverage on security- or data-integrity-sensitive code), call it out explicitly as a recommendation, but don't write code.
>
> End your output with a single JSON object on the last line: `{"coverage_pct": N, "gaps": N, "diagram": "<markdown diagram>"}`. If the diff is test-only or has no application code paths to audit, output `{"coverage_pct": 100, "gaps": 0, "diagram": "No new application code paths — test-only diff."}`.

**Parent processing:**
1. Parse the last line of the subagent's output as JSON.
2. Embed `diagram` verbatim as the PR body's Test Coverage section.
3. Print a one-line summary: `Coverage: {coverage_pct}%, {gaps} gaps.`
4. If `gaps` is high relative to the diff size, or the subagent flagged a serious gap, mention it to the user directly (not just buried in the PR body) — this is the kind of thing worth a human glance before merge, even though it doesn't block Ship itself.

If the subagent fails or times out: note "Coverage audit unavailable this run." in the PR body and continue — never block shipping on this.
