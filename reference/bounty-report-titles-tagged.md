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

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL webhook creation with secret readback | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| CI/CD credentials in public build artifact | GET public artifact URL | Credentials in plaintext | $9,700 |
| Unauthenticated storefront config endpoint leaking live keys | GET config endpoint | Live keys in JSON | $8,000 |
| Cache poisoning via unsanitized header | Unsanitized header | Cache poisoning | $6,000 |
| Alternate vhost / Host header skips SSO | Alternate Host header | Authenticated content | $5,000 |
| Istio RBAC case-sensitivity bypass | Case-varied path | Roster returned | $3,000 |
| Unauthenticated Strapi relational-filter leak → admin takeover | Relational filter | Admin takeover | — |
| Draft auto-save exposing confidential board documents | Draft auto-save | Confidential documents | — |
| Webhook replay and forgery in payment notification | Webhook replay | Payment notification forged | — |
| Arbitrary file read in AI agent tool without path restrictions | File reader without fence | Arbitrary file read | — |
| SSRF via DNS rebinding and URL parser differentials | DNS rebinding / URL parser | Internal response | — |
| GraphQL SSRF and RCE via introspection and mutation | GraphQL introspection + mutation | SSRF / RCE | — |
| IDOR via HTTP method swap | Method change | Data returned | — |
| Vertical privilege escalation on admin endpoints | Call admin API with user token | Admin action | — |
| Cache deception via static extension | Static extension on auth path | Private data cached | — |
| Unauthenticated Firebase full database read/write | Firebase rules | Full DB access | — |
| Wildcard injection on auto-login key | % in key lookup | Key leaked → session | — |
| Unauthenticated realtime channel read/write | WebSocket subscribe | Messages readable | — |
| Missing access control on roster / employee directory | GET /roster | Full roster JSON | — |
| Unauthenticated access to /actuator/env and /heapdump | GET /actuator/* | Secrets / memory | — |
| Unauthenticated access to /debug/pprof | GET /debug/pprof | Process data | — |
| Missing auth on webhook create/list/update | POST/GET webhook | Webhook created | — |
| IDOR on invoices, attachments, documents, orders via ID/UUID swap | ID/UUID swap | Other user data | — |
| Unauthenticated GraphQL query leaking admin data | GraphQL query | Admin data | — |
| Missing rate limit enabling mass enumeration | GET loop | Mass data | — |
| Unauthenticated user enumeration via API | GET with email/param | User exists | — |
| Broken function-level authorization (BFLA) | Call higher-privilege endpoint | Privileged action | — |
| Unauthenticated access to admin metrics / Grafana | GET /metrics | Metrics data | — |
| Password reset token leak in response | Reset flow | Token in response | — |
| Improper authentication via response manipulation | Response manipulation | Requests as other user | — |
| Unauthenticated SQL injection in search/filter/login | SQLi payload | SQL data | — |
| DOM XSS via innerHTML / postMessage | Client-side injection | XSS executed | — |
| Sub-user able to delete company records | Sub-user action | Company deleted | — |
| Unauthenticated testing endpoint exposing internal IP | GET testing endpoint | Internal IP | — |
| MariaDB GRANT PROXY unauthorized authentication changes | GRANT PROXY | Admin takeover | — |
| ACL cache collision privilege inheritance | ACL cache collision | Privilege inheritance | — |
| Unauthenticated access to sensitive employee and customer invoice data | GET invoice/employee endpoint | PII + invoices | — |

## Usage

Agents reference this file and include only the titles relevant to their evaluation class. Prefer high-payout and fully unauthenticated patterns first.
