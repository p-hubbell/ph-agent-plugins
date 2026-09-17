# The Heist: Crew of Thieves

A planning → design → implement crew. Named agents, two audit gates, no silent handoffs.

| Call sign | Role | Invoke |
|-----------|------|--------|
| **The Mastermind** | Thinker — any opening → implementation-ready brief | `/the-heist:mastermind` |
| **The Forger** | Designer — UI opening → implementation-ready design brief | `/the-heist:forger` |
| **The Safecracker** | Implementer — ships the brief, then calls review | `/the-heist:safecracker` |
| **The Lookout** | Auditor subagent — thinking, then the brief | dispatched by Mastermind / Forger |
| **The Cleaner** | Code-review subagent — the diff | dispatched by Safecracker |

Methodology is ported from [gstack](https://github.com/garrytan/gstack) (office-hours, CEO/eng/design plan review, spec interrogation, autoplan principles, pre-landing review) and from this marketplace's graphstack agents (facilitator, strategist, architect, software-engineer, review specialists). This plugin does **not** call gstack binaries.

## Flow

```
opening ──► Mastermind or Forger
                │
                ├─ think (forcing questions, scope mode, architecture lock)
                ├─ Lookout × N  (thinking audit)  ──► act on findings
                ├─ write _docs/heist/brief.md
                ├─ Lookout × N  (brief audit)     ──► act on findings
                ▼
           Safecracker
                ├─ implement the brief
                ├─ Cleaner (+ specialists)        ──► act on findings
                └─ done
```

## State

Written in the **target repo**:

```
_docs/heist/brief.md
_docs/heist/thinking.md      # optional working notes
_docs/heist/audit-log.md
_docs/heist/review-log.md
```

## Install

After this marketplace is added:

```
/plugin install the-heist@ph-agent-plugins
```
