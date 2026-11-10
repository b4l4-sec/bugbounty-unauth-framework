# Coverage Angles Ledger

This file is the shared cross-phase coverage ledger for the pure unauthenticated bug bounty framework.

**Purpose**: Distinguish prior tests from untested hypotheses. Later phases consult this to avoid re-testing the same angle on the same surface and to identify gaps. Do not treat a reference match or hypothesis as a finding. Record only what was actually exercised.

## Rules

- This is a living document updated at the end of each phase (or major step).
- Prefer structured entries over free text.
- Never invent coverage. Use `NOT-TESTED`, `PARTIAL`, `TESTED`, `BLOCKED`, or `OUT-OF-SCOPE`.
- Record the exact surface (host, path, method, parameter) when possible.
- For recon (P0/P1), only record passive or light active discovery; do not record vulnerability tests unless one occurred.
- Handoff to next phase must include a short "coverage delta" summary.

## Angle Catalog (Unauth-focused)

Core vulnerability classes and observation angles relevant to unauthenticated surfaces:

| ID | Angle / Class | Typical Surfaces | Notes |
|----|---------------|------------------|-------|
| INFO-DISC | Information Disclosure | Static files, error pages, JS, headers, directory listings, backups, .git, source maps | Common in recon |
| SUB-TAKE | Subdomain Takeover | CNAME / NS pointing to unclaimed services | Passive + confirmation |
| CORS-MIS | CORS Misconfiguration | API endpoints with `Access-Control-Allow-Origin: *` or reflected origin | Check unauth APIs |
| OPEN-RED | Open Redirect | Login, logout, next=, redirect=, returnUrl parameters | Often unauth |
| SSRF-UNA | SSRF (Unauth) | Image fetchers, URL params, webhooks, oEmbed | Rare but high value |
| XSS-REF | Reflected XSS | Search, error, reflected params | Unauth entry points |
| IDOR-PUB | Public IDOR / BOLA | Object IDs in public APIs, sequential IDs | Check if objects are reachable without auth |
| RATE-LIM | Rate Limiting / Abuse | Login (if present), API, forms, SMS/email triggers | Even unauth endpoints |
| JS-LEAK | Sensitive Data in JS | API keys, internal endpoints, secrets in bundles | Pure recon |
| GRAPHQL | GraphQL Introspection / Unauth Queries | /graphql endpoints | Check if introspection enabled |
| JWT-WEAK | Weak JWT / Unauth Token Issues | Tokens in URLs, cookies set without auth | Rare unauth |
| DEBUG-EXP | Debug / Admin Exposure | /debug, /admin, actuator, swagger, phpinfo | High priority |
| FILE-INC | LFI / Path Traversal | File params, download endpoints | Unauth file handlers |
| RCE-UNA | Unauth RCE / Command Injection | Rare; check template engines, file uploads that process | Only if clearly exposed |
| MISC-CFG | Misconfiguration (headers, cookies, CSP, etc.) | All responses | Baseline |

Add project-specific angles as discovered.

## Entry Format

```
[YYYY-MM-DD] ANGLE-ID | surface | status | notes
```

Example:
```
[2026-11-09] DEBUG-EXP | https://api.example.com/actuator/env | PARTIAL | env exposed, secrets masked
```
