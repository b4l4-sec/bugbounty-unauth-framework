# Bug Bounty Report Titles – Deduplicated Collection

**Source:** Public X.com posts, HackerOne, Bugcrowd, and writeups  
**Updated:** November 9, 2026  
**Total unique titles:** 120+

This is the single source of truth. Agents reference only the rows relevant to their class.

## High-Value Patterns (with known payouts)

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| CI/CD: Live credentials in Public Build Artifact | GET public artifact URL | Credentials in plaintext | $9,700 |
| Hardcoded Credentials: Unauthenticated Storefront Config Endpoint Leaks Live Stripe/AWS/OpenAI keys | GET config endpoint | Live keys in JSON | $8,000 |
| How an Unsanitized Header Generated a $6,000 Bounty via Cache Poisoning | Unsanitized header | Cache poisoning | $6,000 |
| SSRF via XXE: Unauthenticated Redis Exposes Every Broker Session | POST XXE | Redis data returned | $5k+$5k |
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Authenticated content | $5,000 |
| Missing Authentication: Anonymous + Merchant Impersonation | GET/POST without token | Sensitive data | $5,000 |
| Unauthenticated RCE via items Filter | Crafted filter | Command output | $4,950 |
| RCE due to unauthenticated Ray Dashboard | GET dashboard | Dashboard accessible | $3,421 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Roster returned | $3,000 |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Master creds | $3,000 |
| Cross-User IDOR in /api/superadminroutes/{assistantID} | GET other ID | Other user data | $3,000 |
| Keycloak Reset Credentials Email Gate Bypass | Bypass email check | Reset granted | $1,500 |
| Unauthenticated ACH file disclosure | GET file endpoint | ACH content | $1,500 |
| Missing Auth via __AppName Parameter | Param fuzz | Backend action | $1,000 |
| Django DEBUG Page Leaks Neo4j Superuser Password | GET debug page | Credentials visible | $1,000 |
| Unauthenticated Spring Boot Actuator secrets leak | GET /actuator/env | Secrets returned | — |
| Unauthenticated access to /actuator/heapdump | GET /actuator/heapdump | Memory data | — |

## Distinct High-Signal Titles (deduplicated)

1. Unauthenticated GraphQL webhook creation with secret readback
2. CI/CD credentials in public build artifact
3. Unauthenticated storefront config endpoint leaking live keys
4. Cache poisoning via unsanitized header ($6,000)
5. Alternate vhost / Host header skips SSO
6. Istio RBAC case-sensitivity bypass
7. Unauthenticated Strapi relational-filter leak → admin takeover
8. Draft auto-save exposing confidential board documents
9. Webhook replay and forgery in payment notification
10. Arbitrary file read in AI agent tool without path restrictions
11. SSRF via DNS rebinding and URL parser differentials
12. GraphQL SSRF and RCE via introspection and mutation
13. IDOR via HTTP method swap
14. Vertical privilege escalation on admin endpoints
15. Cache deception via static extension
16. Unauthenticated Firebase full database read/write
17. Wildcard injection on auto-login key
18. Unauthenticated realtime channel read/write
19. Missing access control on roster / employee directory
20. Unauthenticated access to /actuator/env and /heapdump
21. Unauthenticated access to /debug/pprof
22. Missing auth on webhook create/list/update
23. IDOR on invoices, attachments, documents, orders via ID/UUID swap
24. Unauthenticated GraphQL query leaking admin data
25. Missing rate limit enabling mass enumeration
26. Unauthenticated user enumeration via API
27. Broken function-level authorization (BFLA)
28. Unauthenticated access to admin metrics / Grafana
29. Password reset token leak in response
30. Improper authentication via response manipulation
31. Unauthenticated SQL injection in search/filter/login
32. DOM XSS via innerHTML / postMessage
33. Sub-user able to delete company records
34. Unauthenticated testing endpoint exposing internal IP
35. MariaDB GRANT PROXY unauthorized authentication changes
36. ACL cache collision privilege inheritance
37. Unauthenticated access to sensitive employee and customer invoice data

## Usage

Agents reference this file and include only the titles relevant to their evaluation class. Prefer high-payout and fully unauthenticated patterns first.
