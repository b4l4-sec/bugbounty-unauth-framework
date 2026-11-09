# Mini-Agent: Cloud & Subdomain Takeover

**Agent ID**: `cloud`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (artifact/image rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| CI/CD: Live credentials in Public Build Artifact | GET public artifact URL | Credentials in plaintext | $9,700 |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Master creds in layers | $3,000 |

## Role

Identify dangling CNAMEs, open storage, and public artifacts containing credentials.

## What to do

1. Check CNAMEs against fingerprints.
2. Look for open storage and public artifacts.
3. Match to the patterns above.
4. Update coverage. Never create resources.
