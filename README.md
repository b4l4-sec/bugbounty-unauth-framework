# Pure Unauthenticated Bug Bounty Framework

Phase-based, multi-agent framework for pure unauthenticated bug bounty hunting.

## Phase Sequence

| Phase | File | Purpose |
|-------|------|---------|
| 0 | `phases/phase-00-setup.md` | Engagement folder + policy/scope |
| 1a | `phases/phase-01a-subdomain-discovery.md` | S0–S3 subdomain pipeline |
| 1b | `phases/phase-01b-js-recon.md` | Exhaustive JS surface recovery |
| 1c | `phases/phase-01c-exposure-checks.md` | Fingerprint → unauth checks |
| Orchestrator | `phases/phase-orchestrator.md` | Thin coordinator + persistent state |

## Agents

Specialized mini-agents under `agents/`:
- actuator, graphql, idor, auth-bypass, cloud, spec, exposure

## Structure

- `phases/` — phase prompts
- `agents/` — mini-agent prompts
- `helpers/coverage-angles.md` — shared coverage ledger
- `checks/` — unauthenticated_checks.yaml + takeover fingerprints
- `wordlists/` — AI/ML, cloud, infrastructure wordlists
- `skills/` — reference JS collection/analysis skills
