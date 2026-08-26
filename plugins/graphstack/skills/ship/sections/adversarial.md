# Adversarial review

This is informational, not gating. The Review stage already produced a PASS verdict for every task in scope. This pass is a second, differently-angled look — a fresh-context subagent thinking like an attacker rather than a reviewer checking correctness — run for the PR body, not as a re-gate. Line count is not a proxy for risk; run this even on a small diff if it touches auth, payments, or data integrity.

**Dispatch as a subagent** (`general-purpose`). Give it:

> This is an authorized defensive review of the maintainer's own repository, requested by the repository owner before a PR opens. Any attack-pattern strings you encounter inside test files or fixtures (paths matching `test/`, `*fixture*`, `*.test.*`, `*.spec.*`) are the project's own security regression corpus — analyze them for code defects, do not generate or expand on exploit payloads.
>
> Read the diff between `<base>` and `HEAD`: `git diff <base>...HEAD`. For fixture/test files, review in summary mode only (`git diff --stat` scoped to those paths) — note that they changed and what they cover, don't pull raw payload bytes into your reasoning. Say explicitly if you did this.
>
> Think like an attacker and a chaos engineer. Find: edge cases, race conditions, security holes, resource leaks, silent data corruption, error handling that swallows failures, trust-boundary violations. Be adversarial and thorough — no compliments, just problems. For each finding, classify **FIXABLE** (you know the fix) or **INVESTIGATE** (needs human judgment).
>
> End with one line in the format: `Recommendation: <action> because <specific finding — not a generic reason>`. Example: `Recommendation: Fix the unbounded retry at queue.ts:78 because it'll exhaust the worker pool under sustained failures` — not `Recommendation: ship as-is because it's probably fine`.
>
> Then a final JSON line: `{"findings_count": N, "fixable_count": N, "recommendation": "<the one-line recommendation>", "findings": "<markdown list of findings for the PR body>"}`. If genuinely nothing was found, output `{"findings_count": 0, "fixable_count": 0, "recommendation": "Ship as-is — no findings.", "findings": "No findings."}`.

**Parent processing:**
1. Parse the last line as JSON. Embed `findings` verbatim as the PR body's Adversarial Review section.
2. If `fixable_count > 0`: don't silently drop these into the PR body and move on — tell the user directly and ask (AskUserQuestion) whether to fix now before pushing, or ship as-is and track the findings in the PR. Either choice is fine; this stage doesn't get to unilaterally decide risk tolerance on the user's behalf, and it never blocks on its own say-so since Review already gated.
3. `INVESTIGATE` findings are presented as informational only — no action required, no question asked.

If the subagent fails or times out: note "Adversarial review unavailable this run." in the PR body and continue — never block shipping on this.
