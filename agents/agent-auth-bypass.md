# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (auth-bypass rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Authenticated content without SSO | $5,000 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Roster returned unauth | $3,000 |
| Missing Auth via __AppName Parameter | Request with __AppName | Backend action succeeded | $1,000 |
| Keycloak Reset Credentials Email Gate Bypass | Bypass email check in reset flow | Reset/session granted | $1,500 |
| Broken Access Control: Ghost Account Grants Universal Login | Login with ghost account | Universal login success | $3,000 |

## Role

Test unauthenticated access control bypasses.

## What to do

1. Baseline blocked endpoints.
2. Apply Host, case, and parameter techniques.
3. Match results to the patterns above.
4. Update coverage.
