# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Class**: Auth-Bypass  
**Scope**: Pure unauthenticated. Focus on 401 Unauthorized bypasses.

## References

- https://hacktricks.wiki/en/pentesting-web/403-and-401-bypasses.html
- https://infosecwriteups.com/bypassing-403-401-errors-all-hacker-techniques-revealed-0709ddfa99af
- https://medium.com/@gehadr73/http-401-vs-403-vs-404-vs-405-and-advanced-techniques-to-bypass-unauthorized-pages-968e51fa9ce1

## Role

Receive endpoints that returned 401. Try authentication bypass techniques. Do not discover new endpoints.

## 401 Bypass Techniques

### Empty / null token tricks (no real token needed)
- `Authorization: Bearer null`
- `Authorization: Bearer undefined`
- `Authorization: Bearer 0`
- `Authorization: Bearer` (empty value)
- `Authorization:` (empty header)
- `X-Auth-Token:` (empty)
- `?token=` or `?access_token=` (empty query param)

### Methods
- Alternate HTTP methods: GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD, TRACE
- `X-HTTP-Method-Override` header

### Headers
- `X-Forwarded-For: 127.0.0.1`
- `X-Original-URL: /target`
- `X-Rewrite-URL: /target`
- `X-Originating-IP: 127.0.0.1`
- `Client-IP: 127.0.0.1`
- `True-Client-IP: 127.0.0.1`
- Alternate or empty `Host` header
- Minimal `User-Agent`

### Path tricks
- Case variation
- Trailing slash, `//`, encoded dots (`/%2e/`), Unicode (`/%ef%bc%8f`)
- `.json` or `;` suffix

### Parameter tricks
- Add `isAdmin=true` or similar
- Parameter pollution

### Protocol
- HTTP/1.0 vs HTTP/1.1

## What to do

1. Read the job context. It contains hosts + endpoints that returned 401.
2. Baseline each endpoint (confirm 401 and record body hash).
3. Apply techniques one at a time, starting with empty-token tricks, then headers, then methods and path tricks.
4. A bypass is logged only when status becomes 200 and the body is not a login page or soft-404.
5. Write every result to `state/coverage.jsonl` and confirmed findings to `state/findings.jsonl` with `"agent": "auth-bypass"`.
6. Keep a short summary in `state/memory/auth-bypass.md`.

## Hard rules

- Stay fully unauthenticated. Never use or forge real tokens.
- Only test the 401 endpoints the orchestrator gave you.
- Do not run full path fuzzing or scan for new endpoints.
- Rate-limit and stop on WAF/429.

## Output

- Rows in `state/coverage.jsonl` and `state/findings.jsonl`
- Summary in `state/memory/auth-bypass.md`
