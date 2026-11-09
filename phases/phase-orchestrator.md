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

### Coverage schema (`coverage.jsonl`)

```json
{
  "timestamp": "2026-11-09T22:15:00Z",
  "agent": "actuator",
  "angle": "DEBUG-EXP",
  "surface": "https://api.example.com/actuator/env",
  "status": "TESTED|PARTIAL|NOT-TESTED|BLOCKED",
  "evidence": "raw/responses/actuator-env-200.txt",
  "notes": "env exposed but secrets masked"
}
```

### Findings schema (`findings.jsonl`)

```json
{
  "id": "F-001",
  "agent": "graphql",
  "host": "api.example.com",
  "title": "Unauth GraphQL introspection",
  "severity": "MEDIUM",
  "evidence": ["raw/responses/graphql-introspection.json"],
  "status": "candidate",
  "created": "2026-11-09T22:20:00Z"
}
```

## Inputs

- `raw/live-hosts.txt`, `raw/scope-hosts.txt` (Phase 1a)
- `raw/js-codex/*/out/endpoints-all.txt`, `orphans.txt`, `secrets.txt` (Phase 1b)
- Fingerprint results and exposure hits from Phase 1c
- `checks/unauthenticated_checks.yaml`
- Filtered RAG slices (only the techniques matching a host’s tech + agent class)

## Steps

### 1. Initialize or resume state

```bash
mkdir -p state/memory state/jobs
touch state/jobs.jsonl state/coverage.jsonl state/findings.jsonl
```

If resuming, read the last 50 lines of each JSONL and `state/orchestrator.md` so the coordinator knows what is already done.

### 2. Build the work inventory

- Load live hosts and high-value endpoints from Phase 1b.
- For each host, match against known fingerprints (from Phase 1c or httpx tech-detect).
- Create one job per (endpoint × relevant agent class).

Agent assignment rules (examples):

| Signal | Agent |
|--------|-------|
| Spring / actuator paths | actuator |
| `/graphql` or GraphQL keywords | graphql |
| Object IDs in responses or paths | idor |
| 401/403 on interesting paths | auth-bypass |
| JS secrets or config | js |
| Cloud bucket / CNAME | cloud |
| OpenAPI / Swagger | spec |
| Default | exposure (uses unauthenticated_checks.yaml) |

### 3. Prepare bounded context for each job

For every job, write a small `context_file` containing only:

- The exact host + endpoint
- Relevant fingerprint
- 5–10 matching techniques from the RAG (filtered)
- Any prior coverage rows for that surface
- Agent-specific memory snippet

Do **not** include full response bodies or entire JS files.

### 4. Dispatch

The orchestrator does not run the agents itself. It either:

- Writes the job files and prints a dispatch list, or
- Calls the appropriate mini-agent prompt with the job id and context path.

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
- Respect Phase 0 scope. Drop any job whose host is out of scope.
- Do not auto-execute state-changing checks.
- Update state atomically (append-only JSONL preferred).

## Handoff

- Mini-agents receive only their job context files.
- Validation / triage phases read `state/findings.jsonl` and `state/coverage.jsonl`.
- Future runs resume from the same state directory.

## Mini-agent contract (for reference)

Every mini-agent must:

1. Read its `context_file` and its own `state/memory/<agent>.md`.
2. Perform only the checks allowed for its class.
3. Write results to the shared JSONL files (or return them for the orchestrator to append).
4. Never modify another agent’s memory.
5. Exit with a clear status (`done`, `blocked`, `partial`).
