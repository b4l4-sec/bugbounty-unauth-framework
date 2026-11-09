# Phase Orchestrator: Unauth Coordinator

**Framework**: Pure Unauthenticated Bug Bounty  
**Role**: Thin coordinator only. Does not perform deep testing.

## Purpose

Turn Phase 1a/1b/1c outputs into discrete jobs for specialized mini-agents. Maintain persistent state. Merge results.

## Steps

### 2. Build the work inventory

- Load all live hosts and discovered endpoints.
- **Every in-scope external-facing response is eligible** (200, 401, 403, 404, 3xx, 5xx).
- Create jobs by matching signals:

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
| WordPress, Drupal, Laravel, Django, Spring, or other CMS fingerprints | framework-recon | Matching hosts |
| Default / no strong signal | exposure | Remaining hosts |

- Multiple signals on one host create multiple jobs.

### 5. Collect results

Agents write to `state/coverage.jsonl` and `state/findings.jsonl` with their own `agent` field. Orchestrator merges and prioritizes.
