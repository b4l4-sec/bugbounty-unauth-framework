# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**References**:
- `reference/bounty-report-titles-tagged.md` (auth-bypass rows only)
- `reference/hackbot-unified-ragv2.json` → techniques where `vuln_class` is Auth-Bypass (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Authenticated content without SSO | $5,000 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Roster returned unauth | $3,000 |
| Missing Auth via __AppName Parameter | Request with __AppName | Backend action succeeded | $1,000 |

## Role

Test unauthenticated access control bypasses. Use RAG bypass_techniques and detection indicators when present.

## What to do

1. Baseline blocked endpoints.
2. Check the RAG for matching Auth-Bypass techniques and apply them.
3. Apply Host, case, and parameter techniques from the bounty patterns.
4. Match results and update coverage.

## Hard rules

- Stay unauthenticated.
- Cover all HTTP methods.
- Prefer RAG techniques when available.
