# Mini-Agent: Cache (Deception & Poisoning)

**Agent ID**: `cache`  
**Class**: CACHE  
**References**:
- `reference/unauth-vectors-extract.md` (Cache rows)
- `reference/bounty-report-titles-tagged.md` (cache deception / poisoning rows)
- Full `vectors-server.md` cache sections

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Cache deception allows unauthenticated access to authenticated user data | Static extension | Private data cached | — |
| How an Unsanitized Header Generated a $6,000 Bounty via Cache Poisoning | Unsanitized header | Cache poisoning | $6,000 |
| Cache deception on /account/settings.css serves private data | Static extension | Private data | — |

## Role

Test cache deception on authenticated paths via static extensions and unkeyed headers.

## What to do

1. Identify candidate authenticated paths from Phase 1b or prior responses.
2. Append static extensions and fetch as anonymous.
3. Check for cache hits containing private data.
4. Match results to bounty patterns and update coverage.

## Hard rules

- Stay unauthenticated for the deception fetch.
- Use cache busters.
