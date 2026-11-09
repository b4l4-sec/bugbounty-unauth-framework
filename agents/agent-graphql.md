# Mini-Agent: GraphQL

**Agent ID**: `graphql`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (GraphQL rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| Unauthenticated GraphQL resolver missing check | GraphQL query | Private fields returned | — |

## Role

Discover GraphQL endpoints and test unauthenticated access. Prioritize webhook mutations and resolver-level gaps.

## What to do

1. Probe common paths with POST/GET.
2. Attempt introspection and test unauthenticated queries/mutations.
3. Match results to the patterns above.
4. Update coverage.
