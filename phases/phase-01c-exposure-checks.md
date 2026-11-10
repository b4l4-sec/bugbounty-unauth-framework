# Phase 1c: Unauthenticated Exposure Checks

**Framework**: Pure Unauthenticated Bug Bounty  
**Cross-phase coverage**: Read `helpers/coverage-angles.md`.  
This phase fingerprints live hosts and runs matching unauthenticated checks from `checks/unauthenticated_checks.yaml`.

## Purpose

For each live host from Phase 1a/1b:

1. Fingerprint the technology (httpx tech-detect + title + headers + known paths).
2. Select the matching check profile from `checks/unauthenticated_checks.yaml`.
3. Probe the listed paths using all relevant HTTP methods.
4. Record positive matches with evidence.
5. Flag high-severity exposures for later validation.

## Inputs

- `raw/live-hosts.txt` / `raw/scope-hosts.txt`
- `checks/unauthenticated_checks.yaml` (fingerprint → paths → matchers)

## Steps

1. Load live hosts.
2. For each host, detect tech stack.
3. Match against profiles in the YAML.
4. Probe unauthenticated paths with GET/POST/OPTIONS as appropriate.
5. Apply matchers and record evidence.
6. Update `state/coverage.jsonl`.

## Output

- Fingerprint results
- Positive exposure candidates with evidence paths
- Coverage updates
- Input for the orchestrator and specialized agents

## Constraints

- Stay unauthenticated.
- Cover all methods on responding endpoints.
- Avoid clearly destructive actions unless explicitly allowed.
