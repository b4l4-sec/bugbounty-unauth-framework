# Mini-Agent: Actuator / Debug

**Agent ID**: `actuator`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (debug/actuator rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated Spring Boot /actuator/env | GET /actuator/env | Secrets in response | — |
| Django DEBUG Page Leaks Neo4j Superuser Password | GET debug/error page | Full credentials on page | $1,000 |
| RCE due to unauthenticated Ray Dashboard | GET dashboard endpoint | Dashboard + RCE possible | $3,421 |

## Role

Probe management and debug endpoints for credential leaks.

## What to do

1. Probe Actuator and equivalent paths with GET/POST.
2. Record credential-exposing responses.
3. Match to the patterns above.
4. Update coverage.
