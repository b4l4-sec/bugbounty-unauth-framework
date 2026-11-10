# Phase Orchestrator: Unauth Coordinator

**Framework**: Pure Unauthenticated Bug Bounty  
**Role**: Thin coordinator. Creates jobs and merges results. Does not probe.

## Handoff rule

Agents do **not** create jobs for other agents.  
Only the orchestrator creates follow-up jobs.

| What an agent sees | What the agent does | What the orchestrator does |
|--------------------|---------------------|----------------------------|
| 200 with interesting content | Writes finding + coverage | Reads finding |
| 403 | Writes coverage row with `"status_code": 403` and `"result": "blocked"` | Creates a new job for `403-bypass` on that exact endpoint |
| 401 | Writes coverage row with `"status_code": 401` | No automatic 403-bypass job (handled differently if needed) |
| Spec found | Writes finding | Creates follow-up jobs based on operations |
| No signal | Writes coverage row | Marks job done |

## 403 → 403-bypass job

When the orchestrator sees:

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
  "id": "job-403-bypass-001",
  "agent": "403-bypass",
  "host": "api.example.com",
  "endpoint": "/webhook",
  "reason": "403 reported by ssrf",
  "status": "pending"
}
```

`403-bypass` receives only the blocked endpoint. `auth-bypass` stays as a separate agent for other auth-related work.

## Steps

1. Create initial jobs from Phase 1 signals.
2. Agents run and write coverage + findings.
3. Orchestrator reads `state/coverage.jsonl`.
4. For every 403 row, create a `403-bypass` job if one does not already exist.
5. Merge findings and write summary.
