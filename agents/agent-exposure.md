# Mini-Agent: General Exposure

**Agent ID**: `exposure`

## Role

Match hosts to YAML profiles and probe listed paths.

## What to do

1. Test the hosts and paths from the job context.
2. If an endpoint returns **401 or 403**, write a coverage row with `"result": "blocked"` and stop for that endpoint.
3. Do not run 403 bypass techniques. The orchestrator will create an auth-bypass job if needed.
4. Continue only on reachable endpoints.
5. Write coverage and findings with `"agent": "exposure"`.

## Hard rules

- Stay unauthenticated.
- Do not perform auth-bypass testing.
- Write a coverage row for every endpoint received.
