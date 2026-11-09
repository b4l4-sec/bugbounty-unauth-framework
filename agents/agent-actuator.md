# Mini-Agent: Actuator / Debug

**Agent ID**: `actuator`  
**References**:
- `reference/bounty-report-titles-tagged.md` (debug/actuator rows only)
- `reference/hackbot-unified-ragv2.json` → techniques related to Actuator, debug, or INFO-DISC (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated Spring Boot /actuator/env | GET /actuator/env | Secrets in response | — |
| Django DEBUG Page Leaks Neo4j Superuser Password | GET debug/error page | Full credentials on page | $1,000 |
| RCE due to unauthenticated Ray Dashboard | GET dashboard endpoint | Dashboard + RCE possible | $3,421 |

## Role

Probe management and debug endpoints for credential leaks. Use RAG detection indicators and sample payloads when present.

## What to do

1. Probe Actuator and equivalent paths with GET/POST.
2. Check the RAG for matching Actuator/debug techniques and apply detection_request / detection_response_indicators.
3. Record credential-exposing responses.
4. Match to the bounty patterns above.
5. Update coverage.

## Hard rules

- Stay unauthenticated.
- Cover all methods on responding paths.
- Prefer RAG techniques when available.
