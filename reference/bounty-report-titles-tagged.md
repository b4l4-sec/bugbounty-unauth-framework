# Bug Bounty Report Titles – Deduplicated Tagged Collection

**Source:** Public X.com posts, HackerOne, Bugcrowd, and writeups  
**Updated:** November 9, 2026  
**Tags:** Detection Method | Tech Stack | Signal | Access Level | Payout

This is the single source of truth. Agents reference only the rows relevant to their class.

## High-Value Unauthenticated Patterns (deduplicated)

| Title | Detection Method | Tech Stack | Signal | Access Level | Payout |
|-------|------------------|------------|--------|--------------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | GraphQL | Webhook created / secret returned | Fully unauth | $9,500 |
| CI/CD: Live credentials in Public Build Artifact | GET public artifact URL | CI/CD | Credentials in plaintext | Fully unauth | $9,700 |
| Hardcoded Credentials: Unauthenticated Storefront Config Endpoint Leaks Live Stripe/AWS/OpenAI keys | GET config/storefront endpoint | Unknown | Live keys in JSON | Fully unauth | $8,000 |
| SSRF via XXE: Unauthenticated Redis Exposes Every Broker Session | POST XXE payload | Java + Redis | Redis data returned | Fully unauth | $5k+$5k |
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Unknown + SSO | Authenticated content without SSO | Auth bypass | $5,000 |
| Missing Authentication: Anonymous + Merchant Impersonation | GET/POST without token | Unknown | Sensitive data / impersonation | Fully unauth | $5,000 |
| Unauthenticated RCE via items Filter | Crafted filter param | Unknown API | Command output / side effect | Fully unauth | $4,950 |
| RCE due to unauthenticated Ray Dashboard | GET dashboard endpoint | Ray (Python) | Dashboard + RCE possible | Fully unauth | $3,421 |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Istio | Roster returned unauth | Auth bypass | $3,000 |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Keycloak | Master creds in layers | Fully unauth | $3,000 |
| Broken Access Control: Ghost Account Grants Universal Login | Login with ghost account | Unknown | Universal login success | Auth bypass | $3,000 |
| Missing Auth in Portal API Exposes Employee Data + Allows Writes | GET/POST without token | Unknown | PII + write success | Fully unauth | $3,000 |
| Cross-User IDOR in /api/superadminroutes/{assistantID} | GET with other ID | Unknown API | Other user’s data | IDOR | $3,000 |
| Keycloak Reset Credentials Email Gate Bypass | Bypass email check in reset flow | Keycloak | Reset/session granted | Auth bypass | $1,500 |
| Unauthenticated ACH file disclosure | GET file endpoint | Unknown | ACH content returned | Fully unauth | $1,500 |
| Missing Auth via __AppName Parameter | Request with __AppName | Unknown | Backend action succeeded | Fully unauth | $1,000 |
| Django DEBUG Page Leaks Neo4j Superuser Password | GET debug/error page | Django + Neo4j | Full credentials on page | Fully unauth | $1,000 |
| Unauthenticated realtime channel read/write | WebSocket subscribe | Realtime (Firebase/etc) | Messages readable | Fully unauth | — |
| Unauthenticated full roster endpoint | GET /roster | Unknown | Full roster JSON | Fully unauth | — |
| Unauthenticated account takeover via wildcard auto-login key | % in key lookup | SQL LIKE | Key leaked → session | Fully unauth | — |
| Unauthenticated GraphQL resolver missing check | GraphQL query | GraphQL | Private fields returned | Fully unauth | — |
| Unauthenticated Spring Boot /actuator/env | GET /actuator/env | Spring Boot | Secrets in response | Fully unauth | — |

## Usage

Agents must reference this file and include **only** the rows that match their evaluation class. Do not duplicate the full table.
