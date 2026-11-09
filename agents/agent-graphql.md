# Mini-Agent: GraphQL

**Agent ID**: `graphql`  
**References**:
- `reference/bounty-report-titles-tagged.md` (GraphQL rows only)
- `reference/hackbot-unified-ragv2.json` → `techniques` where `vuln_class` is GraphQL-related (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| Unauthenticated GraphQL resolver missing check | GraphQL query | Private fields returned | — |

## Role

Discover GraphQL endpoints and test unauthenticated access. Prefer techniques that appear in both the bounty titles and the RAG.

## What to do

1. Probe common paths with POST/GET.
2. Attempt introspection and test unauthenticated queries/mutations.
3. Check the RAG for matching GraphQL techniques (payloads, bypasses, detection indicators) and apply them when available.
4. Match results to the bounty patterns above.
5. Update coverage.

## Hard rules

- Stay unauthenticated.
- Cover all common GraphQL methods and content types.
- Use RAG techniques when they exist; fall back to bounty patterns otherwise.
