# Mini-Agent: IDOR / BOLA

**Agent ID**: `idor`  
**References**:
- `reference/bounty-report-titles-tagged.md` (IDOR rows only)
- `reference/hackbot-unified-ragv2.json` → techniques where `vuln_class` is IDOR or BAC (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Cross-User IDOR in /api/superadminroutes/{assistantID} | GET with other ID | Other user’s data | $3,000 |
| IDOR via HTTP method swap | Method change | Data returned | — |

## Role

Test public object access using observed or synthetic IDs. Apply RAG sample payloads and bypass techniques when they exist.

## What to do

1. Request path templates with different IDs (GET/POST).
2. Check the RAG for matching IDOR techniques and use them.
3. Flag clear response differences.
4. Match to the bounty patterns above.
5. Update coverage.

## Hard rules

- Stay unauthenticated.
- Use only observed or synthetic IDs.
- Prefer RAG techniques when available.
