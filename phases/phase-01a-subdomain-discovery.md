# Phase 1a: Subdomain Discovery Pipeline (Unauth)

**Framework**: Pure Unauthenticated Bug Bounty  
**Cross-phase coverage**: Read `helpers/coverage-angles.md`.  
This phase only discovers and maps hosts. No vulnerability testing, no content discovery beyond HTTP probing, no JS analysis.

## Purpose

Build a bounded, high-signal host inventory for the engagement using a four-stage pipeline.  
Prefer precision over volume. Cap results aggressively. Record everything under `raw/discovery/`.

## Inputs

- Apex / root domains from Phase 0 scope
- Wordlists:
  - `wordlists/ai_ml_automation.txt`
  - `wordlists/cloud_services.txt`
  - `wordlists/infrastructure_common.txt`
- Takeover fingerprints: `checks/takeover_fingerprints.yaml`

## Pipeline Stages

### S0 — Horizontal Enumeration
- SPF/DMARC walking
- Certificate transparency / org search
- TLD expansion
- DNS infrastructure correlation
- Favicon hash, analytics ID, reverse WHOIS

### S1 — Vertical Subdomain Enumeration
- AI/ML + cloud + infrastructure wordlist spray via subfinder/dnsx

### S2 — Cloud Enumeration
- S3/GCS/Azure blob discovery per root

### S3 — HTTP Probing
- Live host detection with httpx

## Output

- `raw/live-hosts.txt`
- `raw/scope-hosts.txt`
- Discovery evidence under `raw/discovery/`
- Coverage updates for recon angles

## Constraints

- Stay inside Phase 0 scope.
- Cap volume aggressively.
- No vulnerability testing in this phase.
