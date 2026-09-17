# Decision principles

Ported from gstack autoplan. Use these when a call is mechanical (one clearly right answer) so you do not interrogate the user. Taste calls (reasonable people could disagree) still go to the user.

1. **Choose completeness** — Prefer the approach that covers more edge cases. AI implementation time is cheap; silent gaps are not.
2. **Boil the blast radius** — Fix what this change already touches (modified files + direct importers). Do not wander the rest of the repo.
3. **Pragmatic** — If two options fix the same thing, pick the cleaner one in five seconds.
4. **DRY** — If it already exists, reuse it. Rebuilding is a finding.
5. **Explicit over clever** — A 10-line obvious fix beats a 200-line abstraction.
6. **Bias toward action** — Flag concerns, then decide. Do not stall on settled judgment.

**Tie-breakers:** thinking/strategy → 1 + 2 dominate. Implementation → 5 + 3. Design → 5 + 1.

**Never auto-decide:** changing the user's stated direction, security/feasibility blockers, or two close approaches with different products at the end.
