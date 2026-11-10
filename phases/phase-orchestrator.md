# Phase Orchestrator: Unauth Coordinator

**Framework**: Pure Unauthenticated Bug Bounty  
**Role**: Thin coordinator. Creates jobs and merges results. Does not probe.

## Purpose

Turn Phase 1 outputs into scoped jobs. Maintain state. Create follow-up jobs from agent results (especially 401/403).

## Handoff rule (no overlap)

Agents do **not** create jobs for other agents.  
Only the orchestrator creates follow-up jobs.

| What an agent sees | What the agent does | What the orchestrator does |
|--------------------|---------------------|----------------------------|
| 200 with interesting content | Writes finding + coverage | Reads finding |
| 401 or 403 | Writes coverage row with `"status_code": 403` and `"result": "blocked"` | Creates a new job for `auth-bypass` on that exact endpoint |
| Spec found | Writes finding | Creates jobs for `idor` / `graphql` / `ssrf` based on operations |
| Object IDs found | Writes finding | Creates job for `idor` |
| No signal | Writes coverage row | Marks job done |

## Job creation for auth-bypass

When the orchestrator sees a coverage row like:

```json
{
  "agent": "ssrf",
  "host": "api.example.com",
  "endpoint": "/webhook",
  "status_code": 403,
  "result": "blocked"
}
```

It creates:

```json
{
  "id": "job-auth-bypass-001",
  "agent": "auth-bypass",
  "host": "api.example.com",
  "endpoint": "/webhook",
  "reason": "403 reported by ssrf",
  "status": "pending"
}
```

`agent-auth-bypass` receives **only** the blocked endpoint. It does not discover new ones.

## Steps

1. Create initial jobs from Phase 1 signals (as before).
2. Agents run and write coverage + findings.
3. Orchestrator reads `state/coverage.jsonl`.
4. For every 401/403 row, create an `auth-bypass` job if one does not already exist for that host+endpoint.
5. Merge findings and write summary.

## Constraints

- Agents never dispatch other agents.
- Auth-bypass only receives endpoints that other agents already marked as blocked.
- Every coverage and finding row must include the `agent` field.
