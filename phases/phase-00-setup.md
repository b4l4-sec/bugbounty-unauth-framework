# Phase 0: Engagement Setup

**Framework**: Pure Unauthenticated Bug Bounty

## Purpose

Create an isolated engagement folder, ingest the program policy and scope, and initialize tracking files. This phase does no testing.

## Steps

1. Create the engagement directory structure:
   ```
   engagement/<program>/
     raw/
     state/
     hunt/
   ```
2. Pull scope and policy (HackerOne API or manual).
3. Write `scope.md` and `policy-notes.md`.
4. Initialize `state/coverage.jsonl`, `state/jobs.jsonl`, and `state/findings.jsonl`.
5. Record any program-specific constraints (rate limits, banned tests, headers).

## Output

- Engagement folder ready
- Scope and policy captured
- Empty state files initialized
- Ready for Phase 1a
