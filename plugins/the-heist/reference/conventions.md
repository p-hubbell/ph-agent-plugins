# The Heist — conventions

State lives in the target repo at `_docs/heist/` (never `~/.gstack` or other global paths). Create the directory on first write.

```
_docs/heist/
  thinking.md          # Mastermind / Forger working notes (optional)
  brief.md             # implementation brief handed to The Safecracker
  audit-log.md         # append-only Lookout findings the lead acted on
  review-log.md        # append-only Cleaner findings the Safecracker acted on
```

Do not call gstack binaries, `~/.gstack` paths, or `$B` / `$D` CLIs. Use this host's tools. If `_docs/graphstack/learnings.md` exists, read it — prior lessons still apply.

## Crew

| Call sign | Job | Writes application code? |
|-----------|-----|--------------------------|
| **The Mastermind** | Think until implementation-ready, then write `brief.md` | No |
| **The Forger** | Same loop, specialized in UI design, then write `brief.md` | No (except optional DESIGN.md if the user asks) |
| **The Safecracker** | Implement `brief.md` | Yes |
| **The Lookout** | Audit thinking or a brief; return findings | No |
| **The Cleaner** | Code-review a diff; return findings | No |

## Dispatching subagents

Prefer plugin agent types `lookout` and `cleaner`. If those types are unavailable, dispatch `generalPurpose` (Cursor) / `general-purpose` (Claude Code) with the full text of `agents/lookout.md` or `agents/cleaner.md` as the brief.

Launch every selected subagent in **one message**, one Agent/Task call each, `run_in_background: false`. Do not do a Lookout's or Cleaner's job inline when a subagent is in scope.

## Question tool

Use `AskQuestion` in Cursor, `AskUserQuestion` in Claude Code. One load-bearing question at a time during discovery. Batched questions are allowed only when the user already handed a detailed spec.

## Findings format (Lookout and Cleaner)

```
- [SEVERITY] (confidence: N/10) where — problem
  Fix: one-line recommended change
  Quote: motivating text or file:line
```

Severity: `CRITICAL` (blocks handoff / must fix) or `INFORMATIONAL`. Confidence 7+ is a real finding; 5–6 must be labeled medium-confidence; below 5 drop from the main list (count only). A finding you cannot quote is unverified — do not report it as fact.
