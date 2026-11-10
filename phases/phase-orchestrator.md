# Phase Orchestrator: Unauth Coordinator

**Framework**: Pure Unauthenticated Bug Bounty  
**Role**: Thin coordinator. Creates jobs and merges results. Does not probe.

## Handoff rule

Agents do **not** create jobs for other agents.  
Only the orchestrator creates follow-up jobs.

| What an agent sees | What the agent does | What the orchestrator does |
|--------------------|---------------------|----------------------------|
| 200 with interesting content | Writes finding + coverage | Reads finding |
| 403 | Writes coverage row with `"status_code": 403` and `"result": "blocked"` | Creates a new job for `403-bypass` |
| 401 | Writes coverage row with `"status_code": 401` and `"result": "blocked"` | Creates a new job for `auth-bypass` |
| Spec found | Writes finding | Creates follow-up jobs based on operations |
| No signal | Writes coverage row | Marks job done |

## 401 → auth-bypass job

When the orchestrator sees:

```json
{
  "agent": "exposure",
  "host": "mail.verisk.com",
  "endpoint": "/api",
  "status_code": 401,
  "result": "blocked"
}
```

It creates:

```json
{
  "id": "job-auth-bypass-001",
  "agent": "auth-bypass",
  "host": "mail.verisk.com",
  "endpoint": "/api",
  "reason": "401 reported by exposure",
  "status": "pending"
}
```

`auth-bypass` receives only the 401 endpoints and applies the techniques from HackTricks and the infosecwriteups article.

## Steps

1. Create initial jobs from Phase 1 signals.
2. Agents run and write coverage + findings.
3. Orchestrator reads `state/coverage.jsonl`.
4. For every 403 row, create a `403-bypass` job if one does not already exist.
5. For every 401 row, create an `auth-bypass` job if one does not already exist.
6. Merge findings and write summary.
