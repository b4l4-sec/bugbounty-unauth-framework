# Server-side, infrastructure, and exposure

Tiers: **L1** confirms the primitive exists. **L2** gets past filters and validation. **L3** uses parser, protocol, and cache-key disagreements between the components in front of the app and the app itself.

---

## SSRF
**Entry points:** webhooks, "import from URL", link previews/unfurls, avatar/image-by-URL, PDF/screenshot generators, RSS/feed readers, file conversion, SVG/XML with external references, OAuth/OIDC `jwks_uri`/`logo_uri`/`request_uri`, integration settings (Jira/Slack/S3 endpoint fields), and `Host`/`X-Forwarded-Host` routing.

**L1 — Basic**
- [ ] **Confirm the server fetches.** Point the input at `https://OAST/`. Record the source IP, the User-Agent, and whether the response body is returned (full-read vs. blind).
- [ ] **Internal reach:**
  ```
  http://127.0.0.1/   http://localhost/   http://[::1]/   http://0.0.0.0/
  http://169.254.169.254/latest/meta-data/                         AWS IMDSv1
  http://169.254.169.254/metadata/instance?api-version=2021-02-01  Azure (needs header Metadata: true)
  http://metadata.google.internal/computeMetadata/v1/              GCP (needs header Metadata-Flavor: Google)
  http://100.100.100.200/latest/meta-data/                         Alibaba
  ```
  GCP and Azure need a request header, so they are only reachable when you control headers (or through a gopher/CRLF primitive). AWS with IMDSv2 enforced needs a PUT for a token first — a plain GET SSRF will fail.

**L2 — Intermediate**
- [ ] IP encodings and wildcard DNS:
  ```
  2130706433   0x7f000001   0177.0.0.1   127.1   127.0.0.1.nip.io
  [::ffff:127.0.0.1]   [0:0:0:0:0:ffff:7f00:1]
  ```
- [ ] Redirects: `https://OAST/302?to=http://169.254.169.254/` (your server redirects; test whether redirects are followed and whether the target is re-validated).
- [ ] Scheme switch: `file:///etc/hostname`, `gopher://`, `dict://` (only if the client library supports them).
- [ ] **PDF/HTML renderers:** inject HTML that the renderer fetches: `<iframe src="http://169.254.169.254/latest/meta-data/">`, `<img src="https://OAST/">`, `<link rel=stylesheet href=…>`; for headless Chrome, a `<script>` that writes `fetch` results into the page.

**L3 — Advanced**
- [ ] **URL parser differentials** between the allowlist check and the HTTP client:
  ```
  http://allowed.com@127.0.0.1/   http://127.0.0.1#@allowed.com/   http://allowed.com\@127.0.0.1/
  http://allowed.com.127.0.0.1.nip.io/
  ```
- [ ] **DNS rebinding:** a hostname that resolves to a public IP at check time and to an internal one at fetch time. It works only when the app resolves twice — once to validate, once in the HTTP client — instead of connecting to the address it validated.
  - Hostname: a public rebinding service such as `rbndr.us` (`<hex-ip-1>.<hex-ip-2>.rbndr.us`, answers alternate between the two with a TTL of 0), or your own authoritative DNS server so you can see every query and the answer it got (`oob_listener.py serve --dns --dns-a <ip>` answers every query with one address and logs each one; without `--dns-a` it resolves nothing).
  - Answers alternate at random, so expect several tries; space them at `RL`. Inputs the server re-fetches (webhooks with retries, feed readers, scheduled imports) give extra resolutions without extra requests from you.
  - What kills it: the client pins the validated IP, or a resolver cache ignores TTL 0 (the JVM, by default, keeps a resolved address for 30 s). One DNS query per fetch in your log → `dead`, with the log as evidence.
  - Proof: as for any SSRF below — internal reach — plus the DNS log showing both answers.
  - Rebinding a victim's own browser against their localhost is a different class and out of scope for a web asset unless the program lists a local agent.
- [ ] Headless renderers: `file://` reads from a page loaded over `file://`; an exposed DevTools endpoint (`http://127.0.0.1:9222/json`) on the rendering host.
- [ ] Header-requiring metadata (GCP, Azure, IMDSv2) through a CRLF-in-URL or gopher primitive that lets you write request headers.
- [ ] Server-side SVG rasterizers (`<image href="http://OAST/">`) and video/document converters that follow external references.
- [ ] Blind SSRF to a single known internal host: response-time or status differences prove reach. Do not sweep internal ranges or ports unless the program explicitly allows it.

**Proof:** the metadata index listing or the IAM role name. If credentials show up in a response, redact them in the report and never use them.

An SSRF that only reaches external hosts — even with a trusted egress IP, leaked trace headers, or a full read of a public page — is usually N/A. Before reporting, prove internal reach: metadata, an internal service's response, a file read, or a protocol-smuggling path. A blind internal callback or timing oracle alone is a lead for L3, not the finding.

## File upload
**L1 — Basic**
- [ ] Executable extension accepted: `.php .php5 .phtml .phar .jsp .jspx .aspx .ashx .cer .shtml`.
- [ ] Served from the app origin with a sniffable type: upload `.html`/`.svg`/`.xml` → stored XSS.
  ```xml
  <svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"/>
  ```
- [ ] Access control on stored files: predictable URLs, files of other users or orgs, files still served after deletion, long-lived signed URLs.

**L2 — Intermediate**
- [ ] Extension tricks: double extension `x.php.jpg`, `x.jpg.php`; case `x.PhP`; trailing `.`/space/`::$DATA` (Windows/IIS).
- [ ] Validation only on `Content-Type` or magic bytes → `GIF89a;` prefix + script content.
- [ ] XXE through SVG, DOCX, XLSX (`vectors-injection.md`).
- [ ] Filename injection: `../../x.txt` (traversal), `"><svg onload=…>.png` (XSS where the name is shown), `x;sleep 5;.jpg` (command injection in processing).
- [ ] Metadata: EXIF GPS not stripped on public images (usually low).

**L3 — Advanced**
- [ ] Server config uploads: `.htaccess` on Apache (`AddType application/x-httpd-php .h1`, then upload `x.h1`); `web.config` on IIS.
- [ ] **Polyglots for re-processing pipelines.** First learn what the pipeline keeps: upload a valid image with a marker in its metadata (an EXIF comment, a PNG `tEXt` chunk), download what the app serves, and compare format, dimensions, metadata, and whether the bytes changed at all. Then aim at what survives:
  - Nothing re-encoded: a file that is a valid image to the validator and HTML or SVG to whatever serves it, where the serving path sniffs content or takes `Content-Type` from the extension (*Type confusion at the edge* below).
  - Metadata kept, pixels re-encoded: the marker in a surviving field reaches whatever later renders or parses metadata (an EXIF viewer, an admin preview) → stored XSS there.
  - Fully re-encoded: the polyglot routes are closed; what remains is the processing library itself (next item).
  - Proof: the marker, or `alert(document.domain)`, in the served file's context, on a file only you and B can see.
- [ ] **Archive extraction** wherever the app unpacks an upload: bulk imports, theme or plugin uploads, backup restores, static-site deploys, Office/EPUB/JAR files it opens as zips.
  - Zip Slip: an entry named `../h1-<username>-<random>.txt` (also `..\` for Windows hosts, and an absolute `/…` name). Most archivers strip `../`, so build the archive with a script (Python `zipfile`/`tarfile`, entry name set explicitly). Detect first without writing outside: the import's file listing, error, or extracted-file URL shows whether the name was kept, stripped, or rejected.
  - A write outside the extraction directory waits for operator approval. With approval: a new file name only, inside a directory the app serves back to you (the parent of the extraction folder, within its own upload area), never an existing file. Fetch it, delete it if the app lets you, and add a Cleanup ledger row.
  - Symlink entries (tar, or zip with Unix attributes) pointing at `/etc/hostname`: if the app serves the extracted file, the response is the server's file → file read with no write. Tar hardlinks work the same way.
  - Zip bombs and huge entry counts are availability-only: out.
- [ ] Processing-library CVEs: ImageMagick (e.g. CVE-2022-44268, PNG `profile` text chunk → file read into the output image), ExifTool (CVE-2021-22204, DjVu), Ghostscript; `ffmpeg` HLS playlists that reference `file://` or internal URLs in video processing. Check the stack version, then use the public detection file for that CVE.
- [ ] Validation race: the file is stored, then checked and deleted — fetch it inside that window (Turbo Intruder, `vectors-logic-api.md`).
- [ ] Type confusion at the edge: the CDN derives `Content-Type` from the extension while the origin stores the declared type, or the reverse.

**Proof of server-side execution:** a file that prints an arithmetic marker (e.g. PHP `<?php echo 7*7; ?>` → `49`), then delete it and say so in the report.

## Path traversal / LFI
**L1 — Basic**
```
../../../../etc/hostname     ..\..\..\windows\win.ini
```
Spots: `file=`, `path=`, `template=`, `lang=`, `download?name=`, export file names, archive members. Proof: `/etc/hostname` or `/etc/passwd`.

**L2 — Intermediate**
```
..%2f..%2f..%2fetc%2fhostname    %2e%2e%2f    ..%252f (double-encoded)
....//....//    /var/www/../../etc/hostname (prefix kept)
```

**L3 — Advanced**
- [ ] Proxy/app normalisation differences: `..;/` (Tomcat/Spring), `%2e%2e/`, encoded slashes the proxy decodes and the app does not.
- [ ] Nginx `alias` off-by-slash: `location /static { alias /var/www/static/; }` → `/static../` reaches the parent directory.
- [ ] Overlong and Unicode encodings on legacy stacks: `%c0%af`, `%u2215`.
- [ ] Traversal into internal API paths built from your input (server-side parameter pollution, `vectors-logic-api.md`).

## HTTP request smuggling / desync
Safety at every tier: never leave a poisoned prefix on a shared connection where another user's request may land; never target endpoints that other people are using in the same window. Confirm with a differential response **on your own follow-up request** (your second request receives the smuggled prefix's response).

**L2 — Intermediate**
- [ ] **Approval first.** The operator writes `sensitive/desync-authorisation.md` (hosts, limits, date). Nothing in this section runs before it exists; `desync_probe.py` checks for it and exits `BLOCKED` otherwise.
- [ ] **Timing detection with the skill's own probe**, one host at a time, every host through `scope_guard` first:
  ```bash
  python3 -I $S/scripts/desync_probe.py --engagement $P --hosts recon/live-hosts.txt \
    --header "$HDR" --rl $RL -v                      # CL.TE, four TE.TE obfuscations, h2c upgrade; TE.CL only with --te-cl
  ```
  Two control POSTs set the baseline; a probe counts when it hangs or exceeds the slowest control by 5 s, and the host stops at its first candidate. Per-host JSON goes to `recon/desync/`. A `void` means the control itself failed and nothing was learned. It never sends a smuggled prefix, so a `CANDIDATE` is the tracker cell `partial` with the JSON as evidence, never a finding: the finding needs a witnessed effect on your own second connection (below).
- [ ] Burp **HTTP Request Smuggler** (Scan → "Smuggle probe"): CL.TE, TE.CL, TE.TE (obfuscated `Transfer-Encoding`). Detection is timing-based; confirm differentially.
- [ ] **Without Burp:** mitmproxy and curl rewrite requests into well-formed HTTP, so neither can send these. `smuggler.py` (defparam; last commit `2be871e`, April 2021) runs timing probes over a raw socket. Checked against that commit:
  - **No header flag** (`-u -v -x -m -l -q -t -c --no-color`). Headers are fixed in two places: `render_template()` in the config file (`configs/default.py`, run with `exec` at start-up) and `_get_cookies()` in `smuggler.py`, the cookie-fetching GET sent before any probe. Add the research header under the `Host` line in both, in your own copy. The fixed Chrome 78 `User-Agent` sits in the same two places.
  - **`-c` path:** the absolute-path check reads the second character, so `-c /full/path.py` is looked up under `configs/`. Put the copy in `configs/` and pass its name, or pass `./<name>.py`.
  - **Order:** each mutation is sent TE.CL first, then CL.TE (`_create_exec_test`) — the reverse of the rule below. Swap the two checks in your copy, or run it only after a CL.TE pass came back clean.
  - **Volume:** `default.py` holds 134 mutations, each probed both ways with a 5 s timeout, and there is no rate-limit flag. Use `-x` (stop at the first hit) and a cut-down config sized to `RL`.
- [ ] **What the timing probe means**, whatever sends it: a body that is complete under one reading of its length (`Content-Length`) and incomplete under the other (`Transfer-Encoding: chunked`) makes whichever server sees it as incomplete wait for bytes that never come. A consistent delay on one variant and a normal answer on its mirror image is the CL.TE or TE.CL signal; a delay on both is the network. Send the CL.TE probe first: on a CL.TE target, the TE.CL probe can leave bytes on the back-end connection that land on someone else's request.

**L3 — Advanced**
- [ ] HTTP/2 downgrade: H2.CL, H2.TE, and CRLF in HTTP/2 header values (`vectors-injection.md`).
- [ ] CL.0: the back end ignores `Content-Length` on some endpoints (static files, redirects, server-level handlers).
- [ ] Client-side desync: the victim's browser sends the desync itself (a `fetch` with a body to an endpoint that ignores it) — no front-end/back-end pair needed.
- [ ] Pause-based desync: the back end times out mid-body and treats the rest as a new request.
- [ ] Newer parser discrepancies (0.CL, `Expect`-based variants; Kettle, 2025) — verify what your current HTTP Request Smuggler build detects.
- [ ] HTTP/2 request-line and pseudo-header injection — `h2.py` is the sender, because every HTTP client normalises these away before they reach the wire: `h2.py alpn` first (if the host does not negotiate h2 the row is honestly `n-a`), then `h2.py send --pseudo ':path=/x HTTP/1.1'`, `--pseudo ':authority=a\r\nX: b'`, a `--header` whose name carries `:` or upper case, and `--content-length` disagreeing with the body for H2.CL. Fields go out as HPACK literals with no Huffman, so the bytes on the wire are the bytes you typed: spaces or CRLF in `:path` or `:method`, a header name containing `:`, or `:authority` and `Host` with different values — the front end reads one, the downgraded HTTP/1.1 request carries another (Kettle, *HTTP/2: The Sequel is Always Worse*, 2021). Same approval as the rest of this section.
- [ ] **h2c upgrade smuggling** (Jake Miller, Bishop Fox, 2020): a reverse proxy that forwards `Upgrade: h2c`, `HTTP2-Settings`, and `Connection: Upgrade, HTTP2-Settings` to a back end that accepts cleartext HTTP/2 becomes a raw tunnel after the `101`; requests inside it reach the back end without the proxy's path rules or auth. Test with `h2csmuggler` (`-x https://<host>/ --test` first; flags vary by version — check `--help`), including over TLS, where the upgrade is non-standard and proxies often pass it through. Prove with one request through the tunnel to a path the proxy normally answers `403`. Same approval as the rest of this section.
- [ ] Impact: capture your own B's request through the queue, or deliver a reflected XSS with no interaction. Stop at one capture.

## HTTP/3 and protocol parity
Edges that speak HTTP/3 run a separate QUIC stack beside their HTTP/1.1 and HTTP/2 listeners. The access rules, path normalisation, and header handling in front of the app are not always identical on each.

**L2 — Intermediate**
- [ ] Find it: `curl -sI -H "$HDR" https://<host>/ | grep -i alt-svc` → `h3=":443"`. Your curl needs HTTP/3 support (`curl -V` lists `HTTP3`).
- [ ] **Parity:** re-send what the edge blocked or rewrote over HTTP/1.1 and HTTP/2 with `curl --http3-only` and the same headers — an edge `403` on a path rule, an anonymous `401`, a path the edge normalised. A protected response that comes back only over HTTP/3 is judged by what it reaches (`reporting.md` Gate 1), not by the protocol.

**L3 — Advanced**
- [ ] Downgrade: HTTP/3 carries the same pseudo-headers as HTTP/2, and the edge rewrites them into HTTP/1.1 for the origin, so the pseudo-header and CRLF checks in *HTTP request smuggling / desync* apply, with the same approval. Few clients send malformed HTTP/3; if yours cannot, leave the cell `partial` and say so.

## Web cache poisoning
**L1 — Basic**
- [ ] Identify caching per path: `X-Cache`, `CF-Cache-Status`, `Age`, `Cache-Control`, `Vary`. Note which static and dynamic paths are cached.
- [ ] Always add a cache buster (`?cb=<random>`) so only your own cache entry is affected.

**L2 — Intermediate**
- [ ] Param Miner → "Guess headers" finds unkeyed inputs: `X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Forwarded-Proto`, `X-Original-URL`, `X-Rewrite-URL`, `X-Host`.
- [ ] Unkeyed header reflected into a script `src`, a redirect, or a resource URL → poisoned response served to others.

**L3 — Advanced**
- [ ] Unkeyed query parameters (`utm_*`), parameter cloaking (`;`), fat GET (body on GET), normalisation differences in the cache key.
- [ ] **Cache key injection** (Kettle, *Web Cache Entanglement*, 2020): learn the key, then collide it.
  - Learn the key with your own cache buster: change one component at a time (path case, each query parameter, `Host`, `Origin`, `Accept-Encoding`, each header named in `Vary`) and watch `HIT`/`MISS` and `Age`. A component that changes the response but not the key is unkeyed (above); a keyed one is a collision candidate.
  - Use what the edge exposes before inferring: Akamai answers `Pragma: akamai-x-get-cache-key` with the key, and Fastly returns debug headers for `Fastly-Debug: 1` where the service allows it.
  - Collision: the cache joins the keyed components into one string with a delimiter. A value in one keyed component that contains that delimiter can make your request's key equal to the key of a different, clean request, so a response built from your input is stored under the clean URL. Prove it with two URLs that both carry your own cache buster: the crafted one, then the clean one that should never have received its response.
- [ ] Poison a resource many pages load (a JS bundle, a JSON config) instead of one HTML page.

**Proof:** two requests with the same cache buster — the poisoned one from you, then a clean one (no injected header, ideally from another IP) receives the poisoned response.

## Web cache deception
**L1 — Basic**
- [ ] Make an authenticated, per-user page look static to the CDN: `/account/profile.css`, `/api/me/x.png`.
- [ ] Visit the URL as B (logged in), then fetch it as anonymous. If B's data comes back from the cache (`X-Cache: HIT`, `Age:`, `CF-Cache-Status: HIT`), it is a finding.

**L2 — Intermediate**
```
/account/profile;.js     /account/profile%23.css    /account/profile%3f.css
```

**L3 — Advanced** — delimiter and normalisation discrepancies between cache and origin (Doyhenart, 2024):
- [ ] Map origin delimiters: which characters make the origin ignore the rest of the path (`;` Spring, `.` Rails format suffix, `%00`, `%0a`), and which the cache treats as literal.
- [ ] Static-directory rules: `/static/..%2faccount/profile` or `/account/profile%2f%2e%2e%2fstatic/x` — one side decodes and resolves the traversal, the other does not.
- [ ] Exact-match static rules: `/account/profile;%2f%2e%2e%2frobots.txt` against a CDN rule that always caches `/robots.txt`.

## Host header attacks
**L1 — Basic**
- [ ] `Host` or `X-Forwarded-Host` reflected into links in responses or emails (password reset poisoning, `vectors-auth.md`).

**L2 — Intermediate**
- [ ] Duplicate `Host` headers; `Host` with a port suffix (`target.com:OAST`).
- [ ] Routing-based SSRF: absolute-URI request line `GET http://internal-host/ HTTP/1.1` with `Host: target.com`, or `Host: 127.0.0.1`.

**L3 — Advanced**
- [ ] Connection-state attacks: the front end validates `Host` only on the first request of a keep-alive connection. Send a valid request, then an internal `Host` on the same connection (Burp Repeater → "Send group in sequence (single connection)").
- [ ] Internal vhost routing: a small list of likely internal names in `Host` against an in-scope front end; stop at the first response that proves internal routing.

## Subdomain takeover
`dns_probe.py takeover --engagement $P --hosts recon/referenced-hosts.txt` is the triage:
it reads CNAME, NS and MX (which `getaddrinfo`, and so `harness.resolves`, cannot see),
fingerprints ~25 providers, and sorts each name into CANDIDATE / look / dead with the
reason. Note the trap it exists for — `harness.py` refuses a host with no DNS answer, and
NXDOMAIN on a name the program still delegates is the *finding*, not a typo; `--unresolved-ok`
is how you send to one once the triage says CANDIDATE. Claiming a name is the operator's
call and no script here does it.

**L1 — Basic**
- [ ] From `resolved.txt`: CNAMEs to S3, Azure (`*.azurewebsites.net`, `*.cloudapp.net`, `*.trafficmanager.net`, `*.blob.core.windows.net`), Heroku, GitHub Pages, Fastly, Shopify, Zendesk, Netlify, etc., whose provider page says the resource does not exist. `nuclei -tags takeover`.

**L2 — Intermediate**
- [ ] Confirm fingerprints against the `can-i-take-over-xyz` list.
- [ ] Dangling NS delegations to unregistered DNS zones, MX to deprovisioned mail services, expired domains referenced in JS or `script src`.

**L3 — Advanced**
- [ ] Second-order takeovers: a dead bucket or domain loaded as a script, stylesheet, or iframe by the main app → script execution on the main origin.
  - List every third-party script and stylesheet host the app loads (`<script src>`, `<link rel=stylesheet>`, `import()` and loader URLs in bundles) and check each for a host someone could claim: an expired domain, an unclaimed bucket, a deleted GitHub Pages site, an npm package served through a public CDN whose name is free to register.
  - **Subresource Integrity:** an `integrity="sha384-…"` attribute makes the browser refuse a changed file, so a claimable host behind a matching hash is not exploitable — check each tag. SRI on a loader does not cover the scripts the loader injects. Missing SRI on its own is N/A on nearly every program; it matters only with a host that can actually be taken.
- [ ] Delegations to cloud DNS zones that were deleted (a new hosted zone on the provider may land on the same name servers).
- [ ] Impact chaining: cookie scope `.target.com`, CORS allowlist, OAuth redirect allowlist, CSP allowlist, postMessage origin allowlist.

**Proof:** claim the resource and serve one static page at an obscure path (e.g. `/h1-<username>.html`) with your H1 username. Do not serve content at `/`, do not set cookies, do not collect traffic. Release it after triage if the program asks.

## Cloud storage
**L1 — Basic**
```bash
aws s3 ls s3://<bucket> --no-sign-request
curl "https://storage.googleapis.com/<bucket>"
curl "https://<acct>.blob.core.windows.net/<container>?restype=container&comp=list"
```
Bucket names come from JS, HTML, CNAMEs, and common patterns (`<company>-prod`, `<company>-assets`, `<company>-backup`). Confirm the bucket belongs to the target before reporting.

**L2 — Intermediate**
- [ ] "Any authenticated AWS user" ACLs: repeat the listing with credentials from your own unrelated AWS account.
- [ ] Write test, only if read is open; delete after:
  ```bash
  aws s3 cp proof.txt s3://<bucket>/h1-<username>-proof.txt --no-sign-request
  ```

**L3 — Advanced**
- [ ] Firebase: Realtime Database/Firestore rules that allow unauthenticated reads/writes (config from the web bundle).
- [ ] Cognito identity pools with unauthenticated access: the pool ID from JS → `aws cognito-identity get-id` + `get-credentials-for-identity` → `aws sts get-caller-identity`, then enumerate what the role can read with read-only calls only.
- [ ] Pre-signed URL and signed-policy flaws: wildcard paths, overlong expiry, signatures that do not cover the object key.

**Proof:** a listing showing sensitive file names; open at most one file to confirm the sensitivity, and redact it.

## Exposed files, debug, and admin surfaces
**L1 — Basic** — request directly on every live host:

| Path / service | What it gives |
|---|---|
| `/.git/HEAD`, `/.git/config` | Source code (`git-dumper` to confirm; report the exposure, do not mine it for creds to use) |
| `/.env`, `config.yml`, `*.bak`, `*.swp` | Secrets |
| `/actuator/env`, `/actuator/heapdump`, `/actuator/mappings` | Spring Boot config, memory dump, routes |
| `/debug/pprof/`, `/debug/vars` | Go internals |
| `/server-status`, `/phpinfo.php` | Internal IPs, paths, env |
| Swagger UI, GraphiQL, Jupyter, Kibana, Grafana, Jenkins, Airflow, Prometheus | Unauthenticated consoles |
| Source maps (`*.js.map`) | Original front-end source (usually informational alone) |

**L2 — Intermediate**
- [ ] The same paths behind path-normalisation tricks when the proxy blocks them (`/actuator;/env`, `/..;/actuator/env`, `/%2e/actuator`). First pass: *401/403 first pass* below.
- [ ] Framework-specific lists once the stack is known (Laravel `/telescope`, `/horizon`; Rails `/rails/info`; Django debug pages; .NET `/elmah.axd`, `/trace.axd`).

**L3 — Advanced**
- [ ] The same surfaces on origin IPs and vhosts found in `recon.md` L3, where the CDN/WAF rules do not apply.

### 401/403 first pass: `access-bypass`
When a path answers an anonymous request with `401` or `403` — here, in `vectors-frameworks.md` (Spring, Express, ASP.NET, front-end proxies), or on any tracker cell left `partial` by a block — the `access-bypass` skill (`~/work/claude-poc2/.claude/skills/access-bypass`, not shipped with this one) runs the generic bypass set in one go: HTTP methods with case variants, path variants (trailing `/`, `//`, `/%2e/`, `/..;/`, `;x`, `.json`, `%00`/`%0a`/`%20` suffixes, uppercase, overlong UTF-8), IP-spoof and URL-rewrite headers (`X-Forwarded-For`, `X-Original-URL`, …), hop-by-hop stripping, and HTTP/1.0 without `Host`. What it adds over Intruder is the false-positive filter: a result counts only if it is a `2xx` (or a redirect away from login) whose body differs from both the blocked response and a soft-404.

Inside the hard gates only:
- **Only where `scope.md` → Rules requires no research header.** The tool has no option for a header, a cookie, or a proxy, so on a program that requires `X-HackerOne-Research` every probe breaks policy — send the same vectors by hand in Burp instead.
- **Anonymous blocks only.** It cannot carry a session, so a `403` that depends on your role stays in Burp with A's session.
- **Automation allowed** (P0 operator question): one URL is up to about 200 requests.
- **Always `--safe`.** The default sends `DELETE`, `PUT`, `MOVE`, `PROPPATCH`, and other write verbs; without a body they still act on whatever sits at that path.

```bash
DELAY=$(awk -v r="$RL" 'BEGIN{print 1/r}')     # --rate is the minimum gap between request starts, in seconds
python3 ~/work/claude-poc2/.claude/skills/access-bypass/scripts/run.py 'https://<in-scope-host>/<blocked-path>' \
  --safe --rate "$DELAY" --concurrency 1 --max-requests 250 \
  --out engagements/<program>/recon/access-bypass-<host>-<date>
```

Then:
- **Each candidate is a lead.** Re-send its `curl` repro by hand in Burp, then follow `phases/P3-sweep.md` → Per cell step 5. It becomes a finding only when the response holds the content the block was protecting (`reporting.md` Gate 1, "so what").
- **No candidates** → `dead` for the vectors it sent, with `report.html` as the evidence. It does not send cookieless segments (`/(S(x))/`), `%3F` truncation, single-character encodings (`/%61dmin`), identity headers carrying a user ID or email, or anything that needs a session: test those by hand or leave the cell `partial`, naming which remain.

## Open backend services
Backend-as-a-service and data stores that answer the internet directly, often with only a public key in front. Proof at every tier: one record, preferably your own B's, plus the listing of names (tables, indices, types) — never a dump.

**L1 — Basic**
- [ ] Request directly on every live host and on hosts referenced in bundles:

  | Path / service | What it gives |
  |---|---|
  | `/v1/graphql`, `/v1/version`, `/console` | Hasura: queries as the unauthenticated role, version, admin console |
  | `https://<ref>.supabase.co/rest/v1/` | Supabase/PostgREST: the OpenAPI table list (where exposed) |
  | `/metrics` | Prometheus exposition from the app: internal hostnames, routes, queue names |
  | `/api/v1/targets`, `/api/v1/status/config` | Prometheus server: scrape targets and internal topology |
  | `/_cat/indices?v`, `/_cluster/health` | Elasticsearch/OpenSearch without authentication |

**L2 — Intermediate**
- [ ] **Supabase row-level security.** The anon key in the web bundle is public by design; the finding is RLS that lets it read rows it should not. Read one row of a user-data table with only the anon key:
  ```bash
  curl -s "https://<ref>.supabase.co/rest/v1/<table>?select=*&limit=1" \
    -H "apikey: <anon-key>" -H "Authorization: Bearer <anon-key>"
  ```
  Then, logged in as A (A's JWT as the bearer token), filter for B's row (`?user_id=eq.<B's id>`). Writes: test only on rows A owns, with a reversible value.
- [ ] **Hasura permissions.** As the unauthenticated role, and then with A's JWT, query each type in the schema with `limit: 1`. Check whether `x-hasura-role` or `x-hasura-user-id` headers sent by the client are honoured — they should only come from the verified JWT.
- [ ] **Elasticsearch.** Index names alone often show the sensitivity (`users`, `orders`, `logs-prod`). Open at most one document to confirm, and redact it.

**L3 — Advanced**
- [ ] Session variables in Hasura permission rules taken from claims the client controls (an unverified or user-editable JWT claim).
- [ ] Supabase `rpc/` functions marked `SECURITY DEFINER` that bypass RLS for the anon role; storage buckets with public read on per-user paths.
- [ ] The same services on origin IPs, staging hosts, and vhosts from `recon.md` L3.

## Secrets in JS, apps, and repos
**L1 — Basic**
- [ ] **Liveness, with a negative control.** Verify each key with the most harmless call that returns *identity*, not data (the `keyhacks` repo lists calls per provider; prefer the identity ones). Then send the same call with one character changed: the two responses must differ the way a valid and an invalid key would. A secret-shaped string with no liveness proof is not a finding. **The provider is not in the program's scope**, so that call needs an operator ruling recorded as an `approved-hosts:` entry in `scope.md` (scope_guard.py), and the sender must name it: `harness.py send --approved-host api.slack.com ...`. It is not scope — the research header is withheld there and an exclusion still beats it — and `curl` outside the harness is not the answer, because it leaves from an unbound address and writes nothing any gate reads.
  ```bash
  aws sts get-caller-identity                                               # AWS: works for any valid key pair, needs no permissions
  curl -s -H "Authorization: Bearer <xoxb-…>" https://slack.com/api/auth.test # Slack
  curl -sI -H "Authorization: token <ghp_…>" https://api.github.com/user      # GitHub classic PAT: scopes in X-OAuth-Scopes
  ```
- [ ] **Authority from metadata only.** Establish what the key can do from what the identity call returned, the documented scopes, and the provider's introspection endpoints. Do not run privileged operations or read customer data to measure it. Report the key, its source, the liveness call with its negative control, and its authority; do not use it further.

**L2 — Intermediate**
- [ ] Separate real findings from keys meant to be public: Stripe publishable `pk_`, Firebase web config, Google Maps keys with referrer restrictions, Algolia search-only keys are not findings by themselves.

**L3 — Advanced**
- [ ] Secrets in historical bundles, mobile builds, public Docker images, and GitHub history (`recon.md` L3) — still live after "removal" from the current build.

## Known CVEs
**L1 — Basic**
- [ ] From fingerprints (`httpx -td`, headers, JS library versions), check the exact versions against public advisories.

**L2 — Intermediate**
- [ ] Report only with a working, non-destructive PoC on the target. A version number alone is N/A.

**L3 — Advanced**
- [ ] Fresh n-days before public templates exist: diff the vendor's fixed and previous versions to locate the patched check, then build a non-destructive detection (`cve-research` skill).

## CI and supply chain
Only when the program lists the repositories, CI, or build pipeline as in scope. Never trigger the target's privileged workflows with a payload, open PRs or issues designed to execute in their CI, or register and publish packages under names they use — unless the program explicitly authorises that test and says how. Prove exploitability in a copy you own: reproduce the workflow in your own repository, show a harmless marker (`echo h1x`) executing there, and cite the exact lines in the target's workflow.

**L1 — Basic**
- [ ] Read `.github/workflows/*.yml` in every in-scope public repository; run `zizmor` and `actionlint` over them.
- [ ] Secrets printed in public Actions logs or build output (verify per the secrets section above).

**L2 — Intermediate**
- [ ] **Script injection:** untrusted event fields interpolated straight into a `run:` step — `${{ github.event.issue.title }}`, `${{ github.event.pull_request.title }}`, `${{ github.event.comment.body }}`, `${{ github.head_ref }}`.
- [ ] **Untrusted checkout in a privileged trigger:** `pull_request_target` or `workflow_run` that checks out the PR head (`ref: ${{ github.event.pull_request.head.sha }}`) and then builds, installs, or runs scripts from it with secrets or a write token available.
- [ ] Self-hosted runners attached to a public repository that runs workflows for fork PRs.

**L3 — Advanced**
- [ ] Artifacts or caches produced by an untrusted-trigger workflow and consumed by a privileged one (artifact or cache poisoning across workflows).
- [ ] Mutable references in privileged workflows: third-party actions pinned to a branch or tag instead of a commit SHA, `curl | sh` installers fetched from domains that are expired or claimable (`subdomain takeover` above). Usually informational alone; report with a concrete path to execution.
- [ ] Internal package names leaked in public manifests, lockfiles, or source maps that do not exist on the public registry: report the leak and the unclaimed name so the vendor can reserve it. Do not claim it yourself.
