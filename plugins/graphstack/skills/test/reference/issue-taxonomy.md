# Issue Taxonomy

Used by the `qa-engineer` agent to classify every defect found while verifying
a task's Acceptance Criteria. Every finding in a `## QA Log` entry gets a
severity and a category — "doesn't work" is not a classification.

This taxonomy is medium-agnostic: it applies whether the criterion under test
is a UI behavior, an API/CLI contract, a data transformation, or a background
job. Skip categories that don't apply to the task (e.g. Visual/Accessibility
for a headless service) rather than forcing a fit.

## Severity Levels

| Severity | Definition | Examples |
|----------|------------|----------|
| **critical** | Blocks a core workflow, causes data loss, or crashes the app/process | Acceptance-critical action errors out or throws, data deleted without confirmation, process exits non-zero on the happy path |
| **high** | Major feature broken or unusable, no workaround | Endpoint returns wrong data, a required flow silently fails, output is corrupted for valid input |
| **medium** | Feature works but with noticeable problems, workaround exists | Slow response (>5s) on a normal case, validation missing but the flow still completes, correct on the common case but wrong on a stated edge case |
| **low** | Minor cosmetic or polish issue | Typo in output/copy, inconsistent formatting, a non-blocking warning in logs |

## Categories

### 1. Functional
- Acceptance criterion's stated behavior doesn't happen, or happens incorrectly
- Wrong output for valid input; correct output for invalid input (should have been rejected)
- Broken links, dead buttons/handlers, endpoints that 404 or error
- State not persisting (data lost on refresh, restart, or re-run)
- Race conditions (double-submit, stale data, non-idempotent operations that should be idempotent)

### 2. Visual/UI (skip for non-UI tasks)
- Layout breaks (overlapping elements, clipped text, horizontal scroll)
- Broken or missing images/assets
- Font/color inconsistencies, misaligned elements
- Dark mode / theme issues

### 3. UX
- Confusing or missing feedback (user/caller can't tell an action succeeded or failed)
- Missing loading/progress indication for a slow operation
- Unclear error messages ("something went wrong" with no actionable detail)
- No confirmation before a destructive action
- Inconsistent interaction or API patterns across the surface touched by this task

### 4. Content
- Typos and grammar errors in user-facing copy, docs, or error messages
- Outdated or incorrect text
- Placeholder / lorem ipsum / TODO text left in
- Wrong labels on buttons, fields, or API response keys
- Missing or unhelpful empty states

### 5. Performance
- Slow responses/operations relative to what the criterion implies is acceptable
- Excessive resource use (memory, network requests, redundant queries) for the operation's scope
- Blocking work that should be async, or vice versa

### 6. Errors/Logs
- Uncaught exceptions or unhandled promise rejections
- Failed requests (4xx/5xx) where the criterion implies success
- Warnings or stack traces in test/build/run output that indicate a real problem, not noise
- Errors swallowed silently where the criterion implies they should surface

### 7. Accessibility (skip for non-UI tasks)
- Missing alt text on meaningful images
- Unlabeled form inputs
- Keyboard navigation broken
- Insufficient color contrast
- Missing or incorrect ARIA attributes where interactive behavior depends on them

## Verification Checklist

Per acceptance criterion, in order:

1. **Locate the behavior.** Find the code path, command, or test that the criterion describes.
2. **Execute, don't assume.** Run the project's actual test suite, a targeted command, or the relevant script — prefer a real run over reading code and inferring it works. Capture the actual output.
3. **Compare actual vs. expected.** The criterion's stated behavior is the expectation; the command/test output is the actual. A criterion only passes if the actual behavior matches, not if the code merely looks correct.
4. **Check for silent failures.** Errors, warnings, and non-zero exit codes in the output that the criterion doesn't excuse.
5. **Classify any gap.** Severity + category from this file, plus what was checked, what happened, and what was expected — specific enough that the engineer doesn't have to re-derive the investigation.
6. **Don't fabricate verification.** If a criterion requires something you can't execute or observe from this environment (e.g. genuine manual/browser interaction), say so explicitly and count it as a FAIL with a note on what's needed — never a silent pass.
