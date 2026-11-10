# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Class**: Auth-Bypass  
**References**:
- `reference/bounty-report-titles-tagged.md` (auth-bypass rows)
- `reference/hackbot-unified-ragv2.json` → Auth-Bypass techniques (if available)

## Role

Test unauthenticated access control bypasses that are not pure 403 path/header tricks.  
This includes alternate vhost / Host header SSO bypass, case-sensitivity issues, parameter-based bypasses, and other auth logic flaws.

403-specific path and header bypasses are handled by the separate `403-bypass` agent.

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Authenticated content without SSO | $5,000 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Roster returned unauth | $3,000 |
| Missing Auth via __AppName Parameter | Request with __AppName | Backend action succeeded | $1,000 |
| Keycloak Reset Credentials Email Gate Bypass | Bypass email check | Reset granted | $1,500 |

## What to do

1. Receive hosts and endpoints from the orchestrator that show auth-related signals (SSO redirects, vhost differences, parameter clues).
2. Test alternate Host / vhost, case variations, and parameter-based bypasses.
3. Write coverage and findings with `"agent": "auth-bypass"`.
4. If you encounter a pure 403 that looks like a path/header issue, write a coverage row and let the orchestrator route it to `403-bypass`.
5. Do not create jobs for other agents.

## Hard rules

- Stay unauthenticated.
- Focus on auth logic bypasses, not generic 403 path tricks.
- Write a coverage row for every endpoint received.
