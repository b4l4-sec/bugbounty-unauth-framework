# Phase 1b: Exhaustive Unauthenticated JavaScript Surface Recovery

**Framework**: Pure Unauthenticated Bug Bounty  
**Cross-phase coverage**: Read `helpers/coverage-angles.md` and `state/coverage.jsonl`.  
This phase collects and analyzes the complete client-side JavaScript surface. It produces evidence-backed endpoints, hosts, secrets candidates, orphans, routes, and parameters. No vulnerability validation or state-changing actions.

## Purpose

Recover every reachable piece of JavaScript (live + historical) and extract a high-fidelity map of:

- API endpoints, routes, and parameters (with methods when recoverable)
- Hosts and service origins referenced by the client
- Secret / credential candidates (redacted)
- Orphan paths and hidden endpoints
- Client-side configuration, GraphQL operations, and auth hints

Every claim must be traceable to collected bytes or captured traffic. Prefer completeness over speed.

## Inputs

- Live hosts and scope from Phase 1a (`raw/live-hosts.txt`, `raw/scope-hosts.txt`)
- Optional shallow routes from earlier observation
- Reference skills (archived):
  - `skills/jshero-SKILL.md` (collection completeness)
  - `skills/js-codex-SKILL.md` (analysis and evidence standards)
- gf patterns (for GitHub / source leakage later; optional in this phase)

## Output layout (per engagement)

```
raw/js-codex/<asset>/
  raw-js/              external / webpack / vite chunks
  inline-js/           inline <script> bodies
  historical-js/       archived JS (waymore or equivalent)
  _meta/               spider reports, urls.jsonl, endpoints.jsonl
  out/                 endpoints-all.txt, params-all.txt, secrets.txt, orphans.txt, hosts.txt
  analysis/            traffic.jsonl (if browser used), claims, trust-map
  ledger.json          persistent per-target endpoint/parameter ledger
  manifest.md
```

## Collection strategy (adapted from jshero)

### Live collection (multi-pass)

- Prefer a real browser (or headless with interaction) over pure curl.
- Run multiple independent passes (default 3) and merge by content hash.
- Force lazy loading: click, scroll, expand menus, type into inputs.
- Fixed deep settings when using a spider: depth 4, high page/click caps, stay on seed host.
- Capture external JS, inline scripts, source maps, and dynamic import() bundles.
- Stop only when a full pass produces zero new JS hashes, endpoints, and routes.

### Historical collection

- Run waymore (or equivalent archive tool) once per domain.
- Download archived JS bodies into `historical-js/`.
- Merge with live collection before extraction.

### Recursive expansion

- Extract endpoints from collected JS.
- Feed newly discovered in-scope page-like URLs back into the spider.
- Repeat until zero new surface (cap iterations).

## Analysis strategy (adapted from js-codex)

Use deterministic extraction first, focused judgment second.

### Required extractions

1. **Endpoints & routes**
2. **Hosts & origins**
3. **Secrets & credentials** (redacted)
4. **Orphans & hidden endpoints**
5. **Parameters**
6. **Client config & auth hints**

### Ledger

Maintain a persistent `ledger.json` per target. Diff every run so only genuinely new surface is reported.

## Constraints

- Pure unauthenticated. No login, no form submission that creates state, no mutation execution.
- Stay inside Phase 0 scope. Record scope gaps instead of probing them.
- Evidence over belief: every endpoint or secret claim must cite file + offset or traffic record.
- Cap context: bulk JS stays on disk; agents receive only summaries, counts, and bounded snippets.

## Handoff

- Endpoint inventory and secret candidates ready for Phase 1c and specialized mini-agents.
- Persistent ledger and coverage updates available for future runs.
- Orchestrator can create jobs from `out/endpoints-all.txt` and `out/orphans.txt`.
