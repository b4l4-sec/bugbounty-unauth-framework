# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Class**: Auth-Bypass  
**References**:
- `reference/bounty-report-titles-tagged.md` (auth-bypass rows only)
- `reference/hackbot-unified-ragv2.json` → Auth-Bypass techniques (if available)
- https://hacktricks.wiki/en/pentesting-web/403-and-401-bypasses.html

## Role

Test unauthenticated access control bypasses on 401/403 endpoints using the techniques from HackTricks and real bounty reports.

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Authenticated content without SSO | $5,000 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Roster returned unauth | $3,000 |
| Missing Auth via __AppName Parameter | Request with __AppName | Backend action succeeded | $1,000 |

## HackTricks 403/401 Bypass Techniques

### Methods
- Alternate HTTP methods: GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD, TRACE, CONNECT
- `X-HTTP-Method-Override` header

### Headers
- `X-Forwarded-For: 127.0.0.1`, `X-Original-URL`, `X-Rewrite-URL`
- `X-Originating-IP`, `Client-IP`, `True-Client-IP`
- Alternate `Host` header (localhost, internal names)
- Remove `Host` header

### Path tricks
- Case variation: `/admin` → `/ADMIN`
- Trailing slash, `//`, `./`, `;`, `.json` suffix
- Encoded dots: `/%2e/`, `/%252e/`
- Unicode: `/%ef%bc%8f`
- `/admin;.css`, `/admin%00`, `/admin%0a`

### Other
- HTTP/1.0 vs HTTP/1.1
- Double encoding
- Parameter pollution on path or query

## What to do

1. Baseline the blocked endpoint (confirm 401/403 and body hash).
2. Apply techniques one at a time from the HackTricks list above.
3. A bypass is only logged when status changes to 200 and the body is not a login page or soft-404.
4. Check the RAG for additional Auth-Bypass techniques and apply them.
5. Record evidence and update coverage.

## Hard rules

- Stay unauthenticated.
- Cover all HTTP methods.
- Rate-limit and stop on WAF/429.
- Prefer HackTricks techniques and RAG when available.

## Output

- Bypass candidates with exact technique and evidence path
- Coverage rows
- Notes for `state/memory/auth-bypass.md`
