# Mini-Agent: General Exposure (YAML-driven)

**Agent ID**: `exposure`  
**Class**: Multiple  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (use only exposure / dashboard / config / Firebase / Grafana / Keycloak rows)

## Role

You match hosts against profiles in `checks/unauthenticated_checks.yaml` and probe the listed paths. Prioritize patterns from the shared bounty collection that are fully unauthenticated.

## Relevant patterns from bounty collection

| Title | Detection Method | Signal | Automatable? |
|-------|------------------|--------|--------------|
| Unauthenticated access to internal Grafana dashboard | GET metrics | Dashboard data | Yes |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Master creds | Semi |
| Hardcoded Credentials: Unauthenticated Storefront Config Endpoint Leaks Live Stripe/AWS/OpenAI keys | GET config endpoint | Live keys in JSON | Yes |
| Unauthenticated Firebase Realtime Database read/write | Firebase rules | Full DB access | Yes |
| RCE due to unauthenticated Ray Dashboard | GET dashboard | Dashboard accessible | Yes |
| CI/CD: Live credentials in Public Build Artifact | GET public artifact | Credentials in plaintext | Yes |

## What to do

1. Match each host to relevant profiles.
2. Probe only the paths listed for that profile, using all responding methods.
3. Apply matchers and record positive hits with evidence.
4. Match results against the relevant patterns above.
5. Update coverage and memory.

## Hard rules

- Stay unauthenticated.
- One profile at a time per host to keep context small.
- Avoid clearly destructive actions unless explicitly allowed.

## Output

- Matched profiles and positive exposures
- Matches to relevant bounty patterns
- Coverage updates
