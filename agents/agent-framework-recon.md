# Mini-Agent: Framework / CMS Recon

**Agent ID**: `framework-recon`

## Role

Run framework-specific unauthenticated checks when a CMS is fingerprinted.

## What to do

1. Test the framework paths from the job context.
2. If an endpoint returns **401 or 403**, write a coverage row and stop for that endpoint.
3. Do not run 403 bypass techniques.
4. Continue only on reachable endpoints.
5. Write every result to `state/coverage.jsonl` and confirmed findings to `state/findings.jsonl` with `"agent": "framework-recon"`.
6. Keep a short summary in `state/memory/framework-recon.md`.

## Hard rules

- Stay unauthenticated.
- Do not perform auth-bypass testing.
- Write a coverage row for every endpoint received.
