# Mini-Agent: Cloud & Subdomain Takeover

**Agent ID**: `cloud`  
**References**:
- `reference/bounty-report-titles-tagged.md` (artifact/image rows only)
- `reference/hackbot-unified-ragv2.json` → techniques related to cloud, takeover, or public artifacts (use if available)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| CI/CD: Live credentials in Public Build Artifact | GET public artifact URL | Credentials in plaintext | $9,700 |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Master creds in layers | $3,000 |

## Role

Identify dangling CNAMEs, open storage, and public artifacts. Use RAG techniques for artifact and image inspection when present.

## What to do

1. Check CNAMEs against fingerprints.
2. Look for open storage and public artifacts.
3. Check the RAG for matching cloud/takeover techniques and apply them.
4. Match to bounty patterns and update coverage.

## Hard rules

- Stay inside scope.
- No resource creation.
- Prefer RAG techniques when available.
