# Mini-Agent: Cache (Deception & Poisoning)

**Agent ID**: `cache`

## Role

Test cache deception on authenticated paths via static extensions.

## What to do

1. Identify candidate paths from the job context.
2. Append static extensions and fetch as anonymous.
3. Check for cache hits containing private data.
4. Write every result to `state/coverage.jsonl` and confirmed findings to `state/findings.jsonl` with `"agent": "cache"`.
5. Keep a short summary in `state/memory/cache.md`.

## Hard rules

- Stay unauthenticated for the deception fetch.
- Use cache busters.
- Write a coverage row for every path tested.
