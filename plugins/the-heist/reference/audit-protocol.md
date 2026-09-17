# Lookout audit protocol

The Lookout never writes application code and never writes `brief.md`. Findings go back to The Mastermind or The Forger, who act on them.

## Modes

Dispatch with exactly one mode:

- **`thinking`** — Audit the lead's reasoning, premises, and proposed solution *before* a brief exists.
- **`brief`** — Audit `_docs/heist/brief.md` (or the draft in the prompt) for executability and accuracy.
- **`design-thinking`** — Same as `thinking`, plus IA, states, slop risk, hard rules.
- **`design-brief`** — Same as `brief`, plus the design-brief sections.

## Parallel split (recommended)

Launch 2–3 Lookouts in one message, each with a different lens:

| Lens | thinking | brief |
|------|----------|-------|
| **Premises** | Evidence vs assertion; status quo; user specificity | Problem/user/wedge still match the thinking |
| **Grounding** | Proposed solution vs actual repo (`path:line`) | Current State citations are real; approach fits existing patterns |
| **Completeness** | Missing failure modes, shadow paths, constraints | AC checkable; tests mapped; no Open Questions that would change code |

Skip a lens that has no surface (e.g. Grounding on a pure copy change).

## Ready vs not

A brief is **not ready** if any CRITICAL finding remains, if any AC is uncheckable, or if Open Questions would change implementation.

## Handback

```
MODE: thinking|brief|design-thinking|design-brief
LENS: premises|grounding|completeness
VERDICT: READY | NOT READY

FINDINGS:
- [SEVERITY] (confidence: N/10) ...

GAPS CLOSED BY LEAD (suggested):
- ...
```
