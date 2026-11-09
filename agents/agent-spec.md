# Mini-Agent: Spec / OpenAPI / Swagger

**Agent ID**: `spec`  
**References**:
- `reference/bounty-report-titles-tagged.md` (webhook/internal rows only)
- `reference/hackbot-unified-ragv2.json` → techniques related to OpenAPI or spec-driven findings (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| Missing auth on webhook create/list/update | POST/GET webhook | Webhook created | — |

## Role

Locate specs using the merged wordlist and flag operations matching high-value patterns. Use RAG techniques for operation analysis when available.

## What to do

1. Probe `wordlists/swagger-openapi-paths.txt`.
2. Parse any spec found and flag webhook/internal/admin operations.
3. Check the RAG for matching techniques related to discovered operations.
4. If none found, escalate to the orchestrator.
5. Update coverage.

## Hard rules

- Stay unauthenticated.
- Prefer RAG techniques when available.
- Always escalate when the wordlist yields nothing.
