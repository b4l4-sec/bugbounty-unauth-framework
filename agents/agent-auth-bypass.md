# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Class**: Auth-Bypass  
**References**:
- `reference/bounty-report-titles-tagged.md` (auth-bypass rows only)
- `reference/hackbot-unified-ragv2.json` → techniques where `vuln_class` is Auth-Bypass (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Authenticated content without SSO | $5,000 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Roster returned unauth | $3,000 |
| Missing Auth via __AppName Parameter | Request with __AppName | Backend action succeeded | $1,000 |
| Keycloak Reset Credentials Email Gate Bypass | Bypass email check in reset flow | Reset/session granted | $1,500 |

## Role

Test unauthenticated access control bypasses. Prioritize patterns from the shared bounty collection that are automatable or semi-automatable.

## What to do

1. Baseline the blocked endpoint (confirm 401/403/SSO redirect and body hash).
2. Apply one technique at a time. Record request + response for every variation.
3. A bypass is only logged when status changes to 200 and the body is not a login page, soft-404, or baseline match.
4. Check the RAG for matching Auth-Bypass techniques and apply them.
5. Save evidence and append to coverage and findings.

## Hard rules

- Stay unauthenticated.
- Cover all HTTP methods.
- Do not brute-force credentials or perform clearly destructive actions.
- Rate-limit and stop on WAF/429.
- Prefer RAG techniques when available.

## Output

- Bypass candidates with exact technique and evidence path
- Matches to relevant bounty patterns
- Coverage rows
- Notes for `state/memory/auth-bypass.md`
