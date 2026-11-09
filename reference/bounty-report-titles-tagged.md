# Bug Bounty Report Titles – Deduplicated Collection

**Source:** Public X.com posts, HackerOne, Bugcrowd, and writeups  
**Updated:** November 9, 2026  
**Total unique titles:** 100+

This is the single source of truth. Agents reference only the rows relevant to their class.

## High-Value Patterns (with known payouts)

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated GraphQL Allows Webhook Creation, Secret Readback, and Tampering | POST GraphQL mutation | Webhook created / secret returned | $9,500 |
| CI/CD: Live credentials in Public Build Artifact | GET public artifact URL | Credentials in plaintext | $9,700 |
| Hardcoded Credentials: Unauthenticated Storefront Config Endpoint Leaks Live Stripe/AWS/OpenAI keys | GET config endpoint | Live keys in JSON | $8,000 |
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
| Unauthenticated Spring Boot Actuator: Credential-Shaped Secrets Leaking from /actuator/env | GET /actuator/env | Secrets returned | — |
| Unauthenticated access to /actuator/heapdump leaks sensitive memory data | GET /actuator/heapdump | Memory data | — |

## Full Deduplicated List (100 titles)

1. Missing access control on realtime channels allows unauthenticated read/write of any data
2. Unauthenticated endpoint exposing full roster data
3. Missing Object Media URL Write Into Account Recording Session (Improper Access Control)
4. Critical unauthenticated account takeover via auto-login key wildcard injection
5. Misconfigured Security Rules / Unauthenticated Access to Firebase
6. Unauthenticated SQL Injection in Simple School Management System 1.0 (staff_delete.php)
7. Unauthenticated Spring Boot Actuator: Credential-Shaped Secrets Leaking from /actuator/env
8. Unauthenticated RCE in Elementor Pro (CVE-2026-32475)
9. Unauthenticated Users Can Access Other Users’ Bug Report Attachments via Broken Access Control
10. Critical Unauthenticated Access to Sensitive Employee and Customer Data Including Invoice Details
11. Unauthenticated Access to NASA SWEHB Admin Pages
12. Information Disclosure in API Endpoint /users (unauthenticated)
13. Unauthenticated Access to MMGIS Webhooks
14. Unauthenticated API Access Allows CAPTCHA Bypass and Mass Spam Submissions on NASA OIG Form
15. Improper Authentication Allows Making Requests as Other Users
16. Unauthenticated User Enumeration via CKAN API (/api/3/action/user_list)
17. Unauthenticated user can upload an attachment to the last updated report draft
18. Unauthorized Access Exposing Sensitive Data
19. IDOR vulnerability allows unauthorized access to team members
20. Simple Access Control — /sys/users/ 403 but /FUZZ returns 200 with PII
21. Unauthenticated IDOR in /api/users/{id} allows reading any user’s profile data
22. Missing authorization on /api/v1/orders/{order_id} exposes other customers’ orders
23. Unauthenticated GraphQL query leaks admin emails and roles
24. Broken access control on file download endpoint allows reading any user’s documents
25. Unauthenticated access to /admin/metrics endpoint exposes internal Prometheus data
26. IDOR in invoice endpoint allows unauthenticated users to view any invoice PDF
27. Missing access control on webhook configuration allows unauthenticated create/update
28. Unauthenticated SSRF via public reference API sharing token bypass
29. Cache deception allows unauthenticated access to authenticated user data
30. Unauthenticated Firebase Realtime Database read/write of all user records
31. Improper access control on /api/me returns other users’ data when ID is swapped
32. Unauthenticated endpoint leaks full employee directory and contact details
33. Missing auth check on /reports/{id}/attachments allows downloading any report file
34. Unauthenticated account takeover via password reset token leak in response
35. IDOR on /api/v1/workspaces/{uuid}/members exposes team member list
36. Unauthenticated access to internal Grafana dashboard via exposed metrics
37. Broken object-level authorization allows reading any user’s private messages
38. Unauthenticated SQL injection in search parameter dumps user table
39. Missing rate limit + unauthenticated API enables mass data enumeration
40. Unauthenticated file upload on profile endpoint leads to stored XSS / RCE
41. Unauthenticated access to /api/v1/user/{id} returns PII of any account
42. Missing access control on realtime subscription channel allows reading any user’s events
43. IDOR in /billing/invoices/{id} lets unauthenticated users download any invoice
44. Unauthenticated GraphQL introspection + query leaks sensitive admin data
45. Broken access control on /files/{file_id} allows downloading any user’s uploaded file
46. Unauthenticated endpoint /api/roster returns full customer list with emails
47. Missing authorization on PATCH /api/users/{id} allows modifying any account
48. Unauthenticated SSRF in image proxy endpoint accesses internal metadata
49. Cache poisoning / deception serves authenticated responses to unauthenticated users
50. Unauthenticated access to Spring Boot /actuator/env leaks credentials
51. IDOR via UUID swap on /api/documents/{uuid} exposes other users’ documents
52. Unauthenticated user enumeration via /api/users?email= endpoint
53. Missing access control on admin panel after SSO redirect
54. Unauthenticated webhook creation allows attacker-controlled data exfiltration
55. Broken access control on /api/v1/orders allows viewing any order details
56. Unauthenticated SQL injection in login form parameter
57. Missing auth on /api/internal/users exposes employee emails and roles
58. IDOR in comment endpoint allows deleting any user’s comments
59. Unauthenticated access to /debug/pprof exposes internal process data
60. Missing object-level auth on /api/v1/teams/{team_id}/members
61. Unauthenticated IDOR on /api/v1/invoices/{id}/pdf downloads any invoice
62. Missing access control on GraphQL mutation allows updating any user’s profile
63. Unauthenticated endpoint leaks AWS credentials via SSRF to metadata service
64. Broken access control on /api/attachments/{id} returns private report files
65. Unauthenticated user list endpoint exposes all registered emails
66. IDOR in /api/v1/projects/{project_id}/settings allows reading other projects
67. Missing authorization check on /api/v1/messages/{message_id}
68. Unauthenticated Firebase rules allow reading entire database
69. Cache deception on /account/settings.css serves private data to unauthenticated users
70. Unauthenticated access to /api/admin/users returns full user table
71. Missing access control on realtime channel subscription
72. IDOR via sequential IDs on /api/orders/{id}
73. Unauthenticated GraphQL query for user { id email role }
74. Broken access control on file download with predictable filenames
75. Unauthenticated password reset endpoint leaks token in response
76. Missing auth on /api/v1/webhooks allows listing all webhook configs
77. IDOR in /api/v1/users/{id}/orders exposes purchase history
78. Unauthenticated SSRF in URL parameter of image fetcher
79. Missing rate limiting on unauthenticated user search endpoint
80. Unauthenticated access to /actuator/heapdump leaks sensitive memory data
81. IDOR on /api/v1/documents/{doc_id}/download
82. Unauthenticated endpoint /api/roster/full returns employee PII
83. Missing access control after changing HTTP method from GET to POST
84. Broken object-level authorization on /api/v1/comments/{id}
85. Unauthenticated access to internal admin dashboard via direct URL
86. IDOR in team invitation endpoint allows joining any team
87. Missing authorization on /api/v1/billing/payment-methods/{id}
88. Unauthenticated SQL injection in filter parameter
89. Cache deception serves /api/me response to unauthenticated requests
90. Unauthenticated access to /api/v1/reports/{id} returns private vulnerability reports
91. Missing access control on /api/v1/notifications/{user_id}
92. IDOR via UUID in /api/v1/files/{uuid}
93. Unauthenticated user enumeration via timing or error messages
94. Broken access control on /api/v1/subscriptions/{id}
95. Unauthenticated endpoint leaks configuration secrets in JSON response
96. Missing auth check on GraphQL admin query
97. IDOR in /api/v1/invoices allows modifying other users’ invoices
98. Unauthenticated access to /debug/vars exposes internal variables
99. Missing access control on /api/v1/audit-logs/{user_id}
100. Unauthenticated Firebase write allows injecting arbitrary data into user records

## Usage

Agents reference this file and include only the titles relevant to their evaluation class.
