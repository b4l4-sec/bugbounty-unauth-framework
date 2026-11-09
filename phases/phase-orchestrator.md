# Phase Orchestrator: Unauth Coordinator

**Framework**: Pure Unauthenticated Bug Bounty  
**Role**: Thin coordinator only. Does not perform deep testing.  
**Cross-phase coverage**: Owns and updates `state/coverage.jsonl` and `state/jobs.jsonl`.

## Purpose

Turn the outputs of Phase 1a (hosts), 1b (JS surface), and 1c (fingerprints) into discrete jobs for specialized mini-agents.  
Maintain persistent state so later runs and agents never lose context.  
Merge agent results into a single coverage and findings view.

The orchestrator never loads full JS bundles, response bodies, or the entire RAG corpus into its context.

## Persistent state layout

```
state/
  jobs.jsonl              # work queue
  coverage.jsonl          # what has been tested
  findings.jsonl          # candidate findings
  memory/                 # per-agent notes
    actuator.md
    graphql.md
    idor.md
    auth-bypass.md
    js.md
    cloud.md
  orchestrator.md         # high-level run notes
```

### Job schema (`jobs.jsonl`)

```json
{
  "id": "job-20261109-001",
  "phase": "1c",
  "agent": "actuator",
  "host": "api.example.com",
  "endpoint": "/actuator/env",
  "method": "GET",
  "status": "pending|running|done|blocked",
  "priority": 1,
  "context_file": "state/jobs/job-20261109-001-context.json",
  "created": "2026-11-09T22:00:00Z"
}
```

## Inputs

- `raw/live-hosts.txt`, `raw/scope-hosts.txt` (Phase 1a)
- `raw/js-codex/*/out/endpoints-all.txt`, `orphans.txt`, `secrets.txt` (Phase 1b)
- Fingerprint results and exposure hits from Phase 1c
- `checks/unauthenticated_checks.yaml`
- Filtered RAG slices

## Steps

### 1. Initialize or resume state

```bash
mkdir -p state/memory state/jobs
touch state/jobs.jsonl state/coverage.jsonl state/findings.jsonl
```

### 2. Build the work inventory

- Load all live hosts and discovered endpoints from Phase 1a/1b/1c.
- **Every in-scope external-facing response is eligible** (200, 401, 403, 404, 3xx, 5xx).
- Do not pre-filter by status code. A 403 or 404 can still be interesting to the right agent.
- Create jobs by matching signals, but fall back to broad jobs when no strong signal exists:

| Signal | Agent | Scope of job |
|--------|-------|--------------|
| Spring / actuator paths or Java fingerprints | actuator | Matching hosts |
| `/graphql` or GraphQL keywords | graphql | Matching hosts |
| Object IDs in responses or paths | idor | Matching hosts + endpoints |
| 401/403 on interesting paths | auth-bypass | Matching hosts |
| Cloud / CNAME / bucket patterns | cloud | Matching hosts |
| OpenAPI / Swagger paths | spec | Matching hosts |
| Webhook / image-proxy / URL params | ssrf | Matching hosts |
| Authenticated-looking paths | cache | Matching hosts |
| Default / no strong signal | exposure | Remaining hosts |

- When a host has multiple signals, create multiple jobs (one per agent).
- When no signal exists, the exposure agent receives the host.

### 3. Prepare bounded context for each job

For every job, write a small `context_file` containing only:
- The exact host list (or single host)
- Relevant endpoints already discovered
- Prior coverage for that surface
- Agent-specific memory snippet

### 4. Dispatch

The orchestrator writes the job files and prints a dispatch list. Agents read their job context and execute.

### 5. Collect results

After agents finish:
- Append their coverage rows to `state/coverage.jsonl`
- Append findings to `state/findings.jsonl`
- Update the job status to `done` or `blocked`
- Append a one-line note to the agent’s memory file

### 6. Coverage report

Write `hunt/orchestrator-summary.md` with jobs created, completed, coverage by agent, new findings, and remaining gaps.

## Constraints

- Never perform the actual probes. Only schedule and merge.
- Keep context small. Bulk data stays on disk.
- Respect Phase 0 scope.
- Update state atomically (append-only JSONL preferred).

## Handoff

- Mini-agents receive only their job context files.
- Every agent writes coverage and findings with its own `agent` field.
- Validation / triage phases read `state/findings.jsonl` and `state/coverage.jsonl`.
- Future runs resume from the same state directory.
