# Mini-Agent: Actuator / Debug / Management Plane

**Agent ID**: `actuator`  
**Class**: DEBUG-EXP, INFO-DISC  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (use only debug/actuator/pprof/DEBUG rows)

## Role

You probe management and debug endpoints. Prioritize patterns from the shared bounty collection that leak credentials and are fully unauthenticated.

## Relevant patterns from bounty collection

| Title | Detection Method | Signal | Automatable? |
|-------|------------------|--------|--------------|
| Unauthenticated Spring Boot Actuator secrets leak | GET /actuator/env | Secrets returned | Yes |
| Django DEBUG Page Leaks Neo4j Superuser Password | GET debug page | Credentials visible | Yes |
| Unauthenticated access to /debug/pprof | GET /debug/pprof | Process data | Yes |
| RCE due to unauthenticated Ray Dashboard | GET dashboard | Dashboard accessible | Yes |
| Unauthenticated access to /admin/metrics | GET /metrics | Metrics data | Yes |

## What to do

1. Probe the full Actuator surface and equivalent paths on every host using GET and POST where the endpoint responds.
2. Record only responses that match known matchers or clearly expose configuration.
3. Save raw responses under `raw/responses/actuator/`.
4. Flag write-capable endpoints for manual review.
5. Match results against the relevant patterns above.
6. Update coverage and `state/memory/actuator.md`.

## Hard rules

- Stay unauthenticated.
- Cover all methods on responding paths.
- Do not perform obviously destructive actions unless program policy and the job both allow it.
- Do not validate extracted secrets against third parties.

## Output

- Positive exposures with evidence paths
- Matches to relevant bounty patterns
- Coverage rows
- Host-specific notes
