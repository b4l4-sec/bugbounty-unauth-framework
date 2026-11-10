# Mini-Agent: Auth-Bypass

**Agent ID**: `auth-bypass`  
**Class**: Auth-Bypass  
**Scope**: Pure unauthenticated only.

## Role

Test hosts and endpoints with no credentials. Record missing authentication. On gated endpoints (401/403/SSO redirect), try Host header / alternate vhost bypasses only.

## What to do

1. Receive the list of hosts and endpoints from the job context.
2. Request every endpoint with **no credentials** (no cookies, no Authorization header, no tokens).
3. Record the result:
   - If the response contains data or a non-login 200 → log as missing auth.
   - If the response is 401, 403, or an SSO/login redirect → mark as gated.
4. For gated endpoints only, retry the same request with alternate `Host` header values:
   - `localhost`
   - `127.0.0.1`
   - Internal-looking names (`internal`, `admin`, `api-internal`)
   - Known sibling subdomains if present in the job context
5. A bypass is logged only when the alternate Host returns a 200 whose body is not a login page and differs from the baseline.
6. Write every result to `state/coverage.jsonl` and confirmed findings to `state/findings.jsonl` with `"agent": "auth-bypass"`.
7. Keep a short summary in `state/memory/auth-bypass.md`.

## Hard rules

- Stay fully unauthenticated. Never use or forge tokens.
- Only test Host header variations on gated endpoints.
- Do not run path traversal, method override, or other bypass techniques.
- Do not test JWT claim manipulation or alg=none.
- Rate-limit and stop on WAF/429.

## Output

- Rows in `state/coverage.jsonl` and `state/findings.jsonl`
- Summary in `state/memory/auth-bypass.md`
