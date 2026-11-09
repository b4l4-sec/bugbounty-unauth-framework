# Mini-Agent: General Exposure

**Agent ID**: `exposure`  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (exposure/config/dashboard rows only)

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Hardcoded Credentials: Unauthenticated Storefront Config Endpoint Leaks Live Stripe/AWS/OpenAI keys | GET config/storefront endpoint | Live keys in JSON | $8,000 |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Master creds in layers | $3,000 |
| RCE due to unauthenticated Ray Dashboard | GET dashboard endpoint | Dashboard + RCE possible | $3,421 |
| Unauthenticated full roster endpoint | GET /roster | Full roster JSON | — |
| Unauthenticated realtime channel read/write | WebSocket subscribe | Messages readable | — |

## Role

Match hosts to YAML profiles and probe for credential or data exposures.

## What to do

1. Match profiles and probe listed paths.
2. Record positive hits.
3. Match to the patterns above.
4. Update coverage.
