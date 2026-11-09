# Bug Bounty Report Titles – Full Tagged Collection (Expanded)

**Source:** Public X.com posts, HackerOne, Bugcrowd, and writeups  
**Compiled:** November 6, 2026  
**Tags:** Detection Method | Tech Stack | Signal | Access Level | Automatable?

This collection is the operational reference for what unauthenticated and access-control bugs actually look like in the wild. Agents should prefer techniques that match high-automatable, high-signal patterns from this list.

## Automatable Legend

- **Yes** — Fully scriptable (unauth endpoints, simple IDOR, public artifacts, actuator, config leaks)
- **Semi** — Needs payload crafting or judgment (XXE, header tricks, image layers, wildcards, flow analysis)
- **No** — Requires account creation or complex human insight

## High-Value Unauthenticated Patterns

| Title | Detection Method | Tech Stack | Signal | Access Level | Automatable? |
|-------|------------------|------------|--------|--------------|--------------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | GraphQL | Webhook created / secret returned | Fully unauth | Yes |
| CI/CD: Live credentials in Public Build Artifact | GET public artifact | CI/CD | Credentials in plaintext | Fully unauth | Yes |
| Hardcoded Credentials: Unauthenticated Storefront Config Endpoint Leaks Live Stripe/AWS/OpenAI keys | GET config endpoint | Unknown | Live keys in JSON | Fully unauth | Yes |
| SSRF via XXE: Unauthenticated Redis Exposes Every Broker Session | POST XXE | Java + Redis | Redis data returned | Fully unauth | Semi |
| Authentication Bypass: Alternate Vhost Skips the SSO Gate | Alternate Host header | Unknown + SSO | Authenticated content | Auth bypass | Semi |
| Missing Authentication: Anonymous + Merchant Impersonation | GET/POST no token | Unknown | Sensitive data | Fully unauth | Yes |
| Unauthenticated RCE via items Filter | Crafted filter | Unknown API | Command output | Fully unauth | Semi |
| RCE due to unauthenticated Ray Dashboard | GET dashboard | Ray | Dashboard accessible | Fully unauth | Yes |
| Istio RBAC Case-Sensitivity Bypass → Employee Roster Dump | Case-varied path | Istio | Roster returned | Auth bypass | Semi |
| Hardcoded Creds in Public Container Image: Keycloak Master Admin Takeover | Image inspect | Keycloak | Master creds | Fully unauth | Semi |
| Missing Auth in Portal API Exposes Employee Data + Allows Writes | GET/POST no token | Unknown | PII + write | Fully unauth | Yes |
| Cross-User IDOR in /api/superadminroutes/{assistantID} | GET other ID | Unknown API | Other user data | IDOR | Yes |
| Keycloak Reset Credentials Email Gate Bypass | Bypass email check | Keycloak | Reset granted | Auth bypass | Semi |
| Unauthenticated ACH file disclosure | GET file endpoint | Unknown | ACH content | Fully unauth | Yes |
| Missing Auth via __AppName Parameter | Param fuzz | Unknown | Backend action | Fully unauth | Yes |
| Django DEBUG Page Leaks Neo4j Superuser Password | GET debug page | Django + Neo4j | Credentials visible | Fully unauth | Yes |
| Missing access control on realtime channels allows unauthenticated read/write | WebSocket subscribe | Realtime | Messages readable | Fully unauth | Yes |
| Unauthenticated endpoint exposing full roster data | GET /roster | Unknown | Full roster JSON | Fully unauth | Yes |
| Critical unauthenticated account takeover via auto-login key wildcard injection | % wildcard | SQL LIKE | Key leaked → session | Fully unauth | Semi |
| Unauthenticated Users Can Access Other Users’ Bug Report Attachments | GET /attachment/{id} | Unknown | Attachment content | Fully unauth | Yes |
| Unauthenticated Spring Boot Actuator secrets leak | GET /actuator/env | Spring Boot | Secrets returned | Fully unauth | Yes |
| GraphQL Authorization Flaws: Missing check in resolver | GraphQL query | GraphQL | Private data | Fully unauth | Yes |
| IDOR via HTTP Method Swap | Method change | Unknown API | Data returned | IDOR | Yes |
| Unauthenticated GraphQL query leaks admin emails and roles | GraphQL query | GraphQL | Admin data | Fully unauth | Yes |
| Unauthenticated access to /admin/metrics | GET /metrics | Unknown | Metrics data | Fully unauth | Yes |
| IDOR in invoice endpoint allows unauthenticated users to view any invoice PDF | GET /invoice/{id} | Unknown | PDF returned | Fully unauth | Yes |
| Missing access control on webhook configuration | POST webhook | Unknown | Webhook created | Fully unauth | Yes |
| Unauthenticated Firebase Realtime Database read/write | Firebase rules | Firebase | Full DB access | Fully unauth | Yes |
| Unauthenticated access to internal Grafana dashboard | GET metrics | Grafana | Dashboard data | Fully unauth | Yes |
| Unauthenticated access to /debug/pprof | GET /debug/pprof | Go/pprof | Process data | Fully unauth | Yes |
| Unauthenticated webhook creation allows attacker-controlled data exfiltration | POST webhook | Unknown | Webhook created | Fully unauth | Yes |
| Missing auth on /api/internal/users | GET /internal/users | Unknown API | Employee data | Fully unauth | Yes |

## Usage by Agents

- **agent-graphql.md** — Prioritize unauthenticated mutations that create webhooks or return secrets; test resolver-level auth gaps.
- **agent-actuator.md** — Actuator `/env`, `/heapdump`, and equivalent debug pages (Django DEBUG, pprof, Ray dashboard).
- **agent-idor.md** — Method swap, attachment/invoice/document IDs, roster and employee directory endpoints.
- **agent-auth-bypass.md** — Alternate vhost/Host header, Istio case-sensitivity, parameter-based auth bypass (`__AppName`).
- **agent-exposure.md** — Grafana, Keycloak, Ray, Firebase, public config endpoints, CI/CD artifacts.
- **agent-spec.md** — Specs that reveal internal or webhook-related operations.
- **agent-cloud.md** — Public build artifacts and container images containing credentials.

## Full Collection Note

The complete 200+ title set with tags is maintained as the source of truth for prioritization. Agents should treat “Automatable? = Yes” + “Fully unauth” as the highest-priority patterns to test first.
