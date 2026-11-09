# Grok Bot Scraping Routine (Unauth Bounty Signals)

**Purpose**: Continuously collect high-signal unauthenticated findings from public X posts by bug hunters and normalize them into the framework’s reference format.

## Bot Responsibilities

1. Monitor X for posts containing keywords: unauth, unauthenticated, actuator, graphql mutation, firebase rules, webhook, cache deception, host header, idor, pprof, debug.
2. Extract post text, linked writeups, and image descriptions.
3. Normalize into the table format used in `reference/bounty-report-titles-tagged.md`.
4. Deduplicate against the existing collection.
5. Append only fully unauthenticated patterns to `reference/incoming-bounty-signals.md`.
6. Notify the orchestrator when new high-payout or novel techniques appear.

## Integration

- The bot writes to the engagement’s reference directory.
- Agents can re-read the updated collection on each run.
