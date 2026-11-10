# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Class**: Auth-Bypass

## Checklist (current framework paths)

### Mapping
- [ ] Auth mechanism identified (JWT / cookie / API key / OAuth) → `state/memory/auth-bypass-map.json`
- [ ] JWT claims analyzed (if JWT)

### JWT Tests (if applicable)
- [ ] `alg=none` tested
- [ ] Algorithm confusion noted for manual check (if RS256)
- [ ] Role/privilege claim manipulation tested
- [ ] Expired token acceptance tested

### Missing Auth
- [ ] All provided endpoints tested without credentials
- [ ] Sensitive endpoints that return data when unauthenticated are recorded

### Bypass Techniques
- [ ] Path traversal bypasses tested
- [ ] Header-based bypasses tested (especially Host header)

### Findings
- [ ] All findings written to `state/findings.jsonl` with `"agent": "auth-bypass"`
- [ ] Coverage rows written to `state/coverage.jsonl`
- [ ] Notes written to `state/memory/auth-bypass.md`

## Role

Identify the auth mechanism on the hosts you receive, then test for missing authentication and common bypasses (path and header, with focus on Host header).

## What to do

1. Map the auth mechanism for each host.
2. Test all provided endpoints without credentials and record those that return data.
3. Test path traversal and header-based bypasses (focus on `Host` header variations).
4. If JWT is used, test `alg=none`, claim manipulation, and expired tokens.
5. Write every result to `state/coverage.jsonl` and confirmed findings to `state/findings.jsonl` using `"agent": "auth-bypass"`.
6. Keep a short summary in `state/memory/auth-bypass.md`.

## Hard rules

- Stay unauthenticated.
- Cover all HTTP methods on the endpoints you receive.
- Do not brute-force credentials.
- Rate-limit and stop on WAF/429.

## Output

- Rows in `state/coverage.jsonl` and `state/findings.jsonl`
- Summary in `state/memory/auth-bypass.md`
