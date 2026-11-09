# Mini-Agent: Command Injection (Unauthenticated)

**Agent ID**: `cmd-injection`  
**Class**: RCE-UNA, CMD-INJ  
**References**:
- `reference/unauth-vectors-extract.md`
- `reference/bounty-report-titles-tagged.md` (RCE / filter / command output rows)
- `reference/hackbot-unified-ragv2.json` (command injection techniques if available)
- Full `vectors-injection.md` OS command injection section

## Relevant patterns

| Title | Detection Method | Signal | Payout |
|-------|------------------|--------|--------|
| Unauthenticated RCE via items Filter | Crafted filter param | Command output / side effect | $4,950 |
| RCE due to unauthenticated Ray Dashboard | GET dashboard endpoint | Dashboard + RCE possible | $3,421 |

## Role

Test unauthenticated endpoints that may pass user input to system commands (filters, diagnostic tools, filename processing, export features, image/PDF converters). Prefer proof of execution via time delay or OAST callback.

## What to do

1. Identify candidate parameters from Phase 1b/1c (filter, order, filename, url, host, diagnostic endpoints).
2. Test classic separators and blind payloads on unauthenticated endpoints:
   - `;id`, `|id`, `$(id)`, `` `id` ``, `%0aid`
   - Blind: `$(sleep 5)`, `;sleep 5;`, OAST callbacks
3. Check the RAG and vector extracts for matching techniques and apply them.
4. Record only clear execution signals (time delay, OAST hit, or command output).
5. Update coverage and escalate confirmed execution to the orchestrator.

## Hard rules

- Stay unauthenticated.
- Prefer time-based or OAST proof over data exfiltration.
- Do not run destructive commands.
- Prefer RAG and vector techniques when available.

## Output

- Candidate endpoints with evidence
- Execution signals
- Coverage updates
