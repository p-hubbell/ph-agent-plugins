# Design passes

Ported from gstack plan-design-review. The Forger runs these on the idea *and* on the brief. Rate 0–10; anything under 8 is a gap to close before claiming ready.

1. **Information architecture** — first / second / third; named page areas; constraint worship (only three things).
2. **Interaction state coverage** — loading, empty, error, success, partial for every UI feature.
3. **User journey & emotional arc** — what they do, feel, and what the spec supports.
4. **AI slop risk** — specific, intentional UI vs generic patterns. See `design-hard-rules.md`.
5. **Visual hierarchy & type** — one visual anchor; scannable headlines; real typefaces.
6. **Density & chrome** — app UI is calm and dense; marketing is compositional, not a dashboard.
7. **Accessibility of the spec** — 44px targets called out, labels not placeholder-only, contrast, visited links, heading proximity.

If a pass has zero findings, write "No issues" and move on. Do not skip a pass because "this is just a settings page."
