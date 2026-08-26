# CHANGELOG entry

1. Check whether `CHANGELOG.md` exists at the repo root.
   - **Exists:** read its header to learn the existing format (heading style, date format, section names). Match it exactly.
   - **Doesn't exist:** ask the user (AskUserQuestion) whether to create one. If yes, start it with a `# Changelog` heading and use the `## [X.Y.Z] - YYYY-MM-DD` / `### Added` / `### Changed` / `### Fixed` / `### Removed` shape below. If no, skip this section entirely and note in the PR body: "No CHANGELOG.md — skipped."

2. **Enumerate every commit being shipped:**
   ```bash
   git log <base>..HEAD --oneline
   ```
   Copy the full list. Count the commits — this is your checklist for step 5.

3. **Read the full diff** to understand what each commit actually changed:
   ```bash
   git diff <base>...HEAD
   ```

4. **Group commits by theme** before writing anything:
   - New capability
   - Bug fixes
   - Removed / dead code
   - Refactoring
   - Infrastructure / tooling / tests

5. **Write the entry**, covering every group from step 4:
   - Insert it right after the file header, dated today.
   - Format: `## [X.Y.Z] - YYYY-MM-DD` (match whatever version Step 6 of the parent skill just wrote).
   - Categorize into `### Added` / `### Changed` / `### Fixed` / `### Removed` as applicable — omit empty categories.
   - **Voice:** lead with what changed from a user's perspective — what they can now do, what's fixed — not implementation detail. Short, concrete bullets.

6. **Cross-check against step 2's commit list.** Every commit must map to at least one bullet. If a commit isn't represented, add it now — don't let anything ship silently unmentioned.

Don't ask the user to describe the changes themselves — infer from the diff and commit history. If something is genuinely ambiguous (e.g. a commit's intent isn't clear from the diff alone), make your best inference and flag it as a note rather than blocking on it.
