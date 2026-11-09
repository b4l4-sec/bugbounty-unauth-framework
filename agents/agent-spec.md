# Mini-Agent: Spec / OpenAPI / Swagger

**Agent ID**: `spec`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (webhook/internal rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| Missing Auth in Portal API Exposes Employee Data + Allows Writes | GET/POST without token | PII + write success | $3,000 |

## Role

Locate specs using the merged wordlist and flag operations matching high-value patterns.

## What to do

1. Probe `wordlists/swagger-openapi-paths.txt`.
2. Parse any spec found and flag webhook/internal/admin operations.
3. If none found, escalate to the orchestrator.
4. Update coverage.
