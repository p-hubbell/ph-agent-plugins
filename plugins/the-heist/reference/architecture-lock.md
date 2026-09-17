# Architecture lock

Ported from graphstack architect + gstack plan-eng-review. Depth tracks risk: a CRUD form is not an auth flow.

Before asking architecture questions, **read at least one piece of evidence** from the repo (Grep/Glob/Read). Cite `path:line`. If genuinely greenfield, say you searched and found nothing.

## Lock in

1. **Boundaries** — components/modules; new coupling and whether it is justified.
2. **Data flow** — for every new flow, happy / nil / empty / error. ASCII is enough.
3. **Edge cases** — user-visible or system-visible; unhandled = gap + one-line fix.
4. **Failure modes** — one realistic production failure per new codepath. Silent failure with no handling and no test is **CRITICAL**.
5. **Test matrix** — per major flow: unit vs integration/E2E; happy/failure/edge. This seeds Acceptance Criteria.
6. **Security surface** — new endpoints, params, file paths, jobs: who can call, what they get, what they can change.
7. **Rollback** — revert, flag, or migration rollback, and how long.

Skip sections that have no surface (copy-only, config-only). Do not manufacture diagrams for flows that do not exist.

## Prime directives (gstack CEO review, condensed)

- Zero silent failures.
- Every error has a name (what triggers it, what the user sees, whether it is tested). Catch-all `Exception` handlers are a smell.
- Observability is scope, not afterthought, when the change can fail in production.
- Everything deferred is written down in Out of Scope or Open Questions.
- Permission to say "scrap it and do this instead" — say it now, not after the Safecracker starts.
