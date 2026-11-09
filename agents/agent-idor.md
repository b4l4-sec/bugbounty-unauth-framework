# Mini-Agent: IDOR / BOLA (Public Object Access)

**Agent ID**: `idor`  
**Class**: IDOR-PUB  
**Shared reference**: `reference/bounty-report-titles-tagged.md` (use only IDOR / method-swap / object-ID rows)

## Role

You test whether object identifiers are reachable without authentication. Prioritize patterns from the shared bounty collection that are fully unauthenticated and automatable.

## Relevant patterns from bounty collection

| Title | Detection Method | Signal | Automatable? |
|-------|------------------|--------|--------------|
| Cross-User IDOR in /api/superadminroutes/{assistantID} | GET other ID | Other user data | Yes |
| IDOR via HTTP Method Swap | Method change | Data returned | Yes |
| IDOR in invoice endpoint allows unauthenticated users to view any invoice PDF | GET /invoice/{id} | PDF returned | Yes |
| Missing authorization on /api/v1/orders/{order_id} | GET other ID | Order data | Yes |
| IDOR on /api/v1/workspaces/{uuid}/members | UUID swap | Member list | Yes |
| IDOR in /billing/invoices/{id} | GET other ID | Invoice data | Yes |
| Broken access control on /files/{file_id} | GET other ID | File content | Yes |
| Unauthenticated Users Can Access Other Users’ Bug Report Attachments | GET /attachment/{id} | Attachment content | Yes |
| Improper access control on /api/me returns other users’ data | ID swap | Other user data | Yes |

## What to do

1. Collect IDs already observed (int, uuid, mongo ObjectId, opaque).
2. For each path template, request with two different IDs using GET and POST.
3. Flag as candidate when both return 200, bodies differ, and structure is similar.
4. Test `/me`, `/api/me`, `/api/v1/me` with parameter injection.
5. Match results against the relevant patterns above.
6. Update coverage and memory.

## Hard rules

- Stay unauthenticated.
- Use only observed or synthetic IDs.
- Do not enumerate large ranges.
- Cover GET and POST where the endpoint responds.

## Output

- Candidate IDOR surfaces with evidence
- Matches to relevant bounty patterns
- Coverage updates
- Notes for `state/memory/idor.md`
