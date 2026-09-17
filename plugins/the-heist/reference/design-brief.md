# Design brief template

The Forger writes `_docs/heist/brief.md` using the implementation brief shape in `implementation-brief.md`, **plus** these sections so the Safecracker can build UI without inventing taste.

```markdown
## Memorable Thing
One sentence: what someone should remember after the first screen.

## Surface Classifier
MARKETING / APP UI / HYBRID — and which hard-rule set applies.

## Information Architecture
What the user sees first, second, third. ASCII of page structure and nav. If you can only show three things, which three?

## Interaction States
| Feature | Loading | Empty | Error | Success | Partial |
Describe what the user SEES, not backend behavior. Empty states are features (warmth, primary action, context).

## User Journey
| Step | User does | User feels | Spec |
5-second visceral, 5-minute behavioral, 5-year reflective — one line each.

## Visual System
Typefaces (no default stacks as primary), color CSS variables, spacing scale, density, motion (2–3 intentional motions or explicitly none for dense app UI).

## Hard-Rule Check
Litmus YES/NO from `design-hard-rules.md`. Instant-fail patterns named if they were tempting. AI-slop blacklist items explicitly rejected.

## Copy
Product language. If deleting 30% of the copy improves it, it is already too long — cut in this brief.
```

The Safecracker implements this brief; they do not run a second design exploration unless the brief says a token or layout is TBD — and TBD is not allowed in a ready brief.
