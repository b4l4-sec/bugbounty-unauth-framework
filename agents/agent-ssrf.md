# Mini-Agent: SSRF (Unauthenticated)

**Agent ID**: `ssrf`  
**Class**: SSRF-UNA  
**References**:
- `reference/unauth-vectors-extract.md` (SSRF rows)
- `reference/bounty-report-titles-tagged.md` (SSRF / XXE / metadata rows)
- `reference/hackbot-unified-ragv2.json` (SSRF techniques if available)
- Full `vectors-server.md` and `vectors-injection.md` SSRF / XXE sections

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| SSRF via XXE: Unauthenticated Redis Exposes Every Broker Session | POST XXE | Redis data returned | $5k+$5k |
| Unauthenticated SSRF in image proxy endpoint | URL param | Internal response | — |
| Unauthenticated endpoint leaks AWS credentials via SSRF to metadata | URL param | Metadata returned | — |

## Role

Test unauthenticated URL fetch, webhook, image proxy, and XXE endpoints for internal reach. Prefer metadata and internal service responses.

## What to do

1. Identify candidate endpoints from Phase 1b/1c.
2. Confirm the server fetches by pointing at an OAST host.
3. Test internal reach only when the endpoint is unauthenticated.
4. Check the RAG and vector extracts for matching techniques and apply them.
5. Record evidence and update coverage.
6. Escalate high-confidence internal reach to the orchestrator.

## Hard rules

- Stay unauthenticated.
- Stop at proof of internal reach.
- Prefer RAG and vector techniques when available.
