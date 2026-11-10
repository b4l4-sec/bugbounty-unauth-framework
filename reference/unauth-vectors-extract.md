# Unauthenticated Vectors Extract

**Source:** Extracted from docs-mining.md, vectors-auth.md, vectors-client.md, vectors-frameworks.md, vectors-injection.md, vectors-logic-api.md, vectors-server.md  
**Focus:** Pure unauthenticated and pre-auth patterns only.  
**Updated:** November 9, 2026

This file is the filtered unauth slice of the vector collection. Agents reference only the sections relevant to their class.

## High-Priority Unauth Patterns

| Class | Technique | Detection Method | Signal | Source |
|-------|-----------|------------------|--------|--------|
| GraphQL | Unauthenticated mutation (webhook creation, secret readback) | POST GraphQL mutation | Webhook created / secret returned | vectors-logic-api.md |
| GraphQL | Introspection + resolver missing auth | Introspection + query | Private data returned | vectors-logic-api.md |
| Actuator / Debug | /actuator/env, /heapdump, Django DEBUG, pprof | GET debug paths | Secrets / memory / credentials | vectors-frameworks.md, vectors-server.md |
| Firebase | Misconfigured rules allowing unauth read/write | Firebase rules | Full DB access | vectors-server.md |
| IDOR | Method swap (GET blocked, POST open) | Method change | Data returned | vectors-auth.md, vectors-logic-api.md |
| IDOR | UUID / sequential ID swap on public endpoints | ID/UUID swap | Other user data | vectors-auth.md |
| Auth-Bypass | Alternate vhost / Host header skips SSO | Alternate Host | Authenticated content | vectors-server.md |
| Auth-Bypass | Path normalization / case sensitivity (Istio, Spring, Express) | Case or encoded path | Roster or admin data | vectors-frameworks.md |
| Cache | Cache deception on authenticated paths via static extension | Static extension | Private data cached | vectors-server.md |
| SSRF | Unauthenticated URL fetch (webhooks, image proxy, XXE) | URL param or XXE | Internal response | vectors-server.md, vectors-injection.md |
| Specs | OpenAPI / Swagger exposing internal or webhook operations | Spec paths | Operation list | docs-mining.md |
| Exposure | Open Grafana, Elasticsearch, Jupyter, Ray dashboard | GET console paths | Dashboard or data | vectors-server.md |
| Config | Unauthenticated config endpoint leaking live keys | GET config | Live keys in JSON | bounty titles |

## Usage

Agents must reference this file and the full vector files. Prefer techniques that are fully unauthenticated and match high-signal bounty patterns.
