# Mini-Agent: 403 Bypass

**Agent ID**: `403-bypass`

## Role

Receive only endpoints that other agents marked as 403. Try 403 bypass techniques on those endpoints.

## What to do

1. Read the job context (exact host + endpoint that returned 403).
2. Apply path, header, and method bypass techniques.
3. Log a bypass only when status becomes 200 and the body is not a login page.
4. Write every result to `state/coverage.jsonl` and confirmed findings to `state/findings.jsonl` with `"agent": "403-bypass"`.
5. Keep a short summary in `state/memory/403-bypass.md`.

## Hard rules

- Stay unauthenticated.
- Only test the 403 endpoints given by the orchestrator.
- Do not scan for new paths.
