# Mini-Agent: IDOR / BOLA

**Agent ID**: `idor`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (IDOR rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Cross-User IDOR in /api/superadminroutes/{assistantID} | GET with other ID | Other user’s data | $3,000 |

## Role

Test public object access using observed or synthetic IDs.

## What to do

1. Request path templates with different IDs (GET/POST).
2. Flag clear response differences.
3. Match to the patterns above.
4. Update coverage.
