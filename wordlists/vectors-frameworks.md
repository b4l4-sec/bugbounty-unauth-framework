# Frameworks and platforms

The other vector files test classes of bug. This one tests stacks: the defaults, conventions, and known weak spots of the framework or platform a host runs. Run a section only when `map.md` → Frameworks records a fingerprint for it; otherwise its tracker row is `n-a`.

Tiers: **L1** confirms the stack and checks what it ships with by default. **L2** tests the places where the framework's conventions leave authorization to the developer. **L3** goes after disagreements between the framework's router, its security layer, and the proxy in front of it, plus version-specific advisories.

**Known CVEs** in this file follow `vectors-server.md` → Known CVEs: fingerprint the exact version, confirm it falls inside the advisory's affected range, prove with the vendor's or a public non-destructive detection, and stop. A version string alone is N/A. Never run a command-execution payload unless the program asks for RCE proof, and then only `id`/`hostname`. Version ranges below were taken from the advisories when this file was written — re-read the advisory before relying on one.

**Anonymous `401`/`403` blocks** in this file (Spring, Express, ASP.NET, front-end proxies) get the automated first pass in `vectors-server.md` → *401/403 first pass* before the stack-specific variants, within its limits: no required research header, anonymous blocks only, always `--safe`.

---

## Fingerprints
| Stack | Signals |
|---|---|
| Next.js | `/_next/static/`, `X-Powered-By: Next.js`, `__NEXT_DATA__` (pages router), `self.__next_f.push` (app router / React Server Components), `x-nextjs-cache`, `Vary: RSC, Next-Router-State-Tree`; version from `next.version` in the browser console |
| Nuxt | `/_nuxt/`, `window.__NUXT__` or `<script id="__NUXT_DATA__">` |
| SvelteKit | `/_app/immutable/`, `__sveltekit_*` globals |
| Remix / React Router 7 | `window.__remixContext` / `window.__reactRouterContext` |
| Spring | Whitelabel Error Page, JSON errors with `timestamp`/`status`/`error`/`path`, `/actuator`, `JSESSIONID` |
| Rails | `X-Runtime`, `X-Request-Id`, `_<app>_session` cookie, `csrf-token` meta tag, `/rails/active_storage/` |
| Laravel | `laravel_session` and `XSRF-TOKEN` cookies (base64 JSON with `iv`, `value`, `mac`), Ignition error pages, Livewire `wire:` attributes |
| Django / DRF | `csrftoken` and `sessionid` cookies, `csrfmiddlewaretoken` fields, `/static/admin/`, DRF browsable API (`?format=api`) |
| Flask | `session` cookie that base64-decodes to JSON (`flask-unsign --decode`), Werkzeug tracebacks |
| Express / NestJS | `X-Powered-By: Express`, `connect.sid`, `Cannot GET /x`; NestJS errors `{"statusCode":404,"message":"Cannot GET /x",…}` |
| ASP.NET | `X-AspNet-Version`, `ASP.NET_SessionId`, `.AspNetCore.*` cookies, `__VIEWSTATE`, `/_blazor`, `/_framework/` |
| Symfony | `/_profiler`, `/_wdt`, `X-Debug-Token` header |
| WordPress | `/wp-content/`, `/wp-json/` (lists REST namespaces), `<meta name="generator" content="WordPress x.y">`, `?ver=` on core assets |
| Drupal | `X-Generator: Drupal <major>` header or generator meta tag, `/core/misc/drupal.js`, `/sites/default/files/` |
| FastAPI | `/docs` (Swagger UI), `/redoc`, `/openapi.json`; validation errors `{"detail":[{"loc":…,"msg":…,"type":…}]}`; `server: uvicorn` |
| Go (net/http, Gin, Echo, Fiber, Chi) | Plain-text `404 page not found` (net/http, Gin), Echo `{"message":"Not Found"}`, Fiber `Cannot GET /x` (looks like Express), `/debug/pprof/`, `/debug/vars` |
| Phoenix (Elixir) | `_<app>_key` session cookie starting `SFMyNTY.`, `data-phx-*` attributes, `/live/websocket` |
| Firebase / Cognito / Auth0 / Keycloak | Bundle config: `apiKey` + `authDomain`/`projectId`; `UserPoolId` + `ClientId`; `*.auth0.com` or a custom Auth0 domain; `/realms/<realm>/` |
| Salesforce Experience Cloud | `/s/sfsites/aura`, `/s/sfsites/auraFW/`, `*.my.site.com` |
| ServiceNow | `/api/now/`, `/sp`, `glide_*` cookies |

Record stack, version, and the evidence per host in `map.md` → Frameworks. `httpx -td` and Wappalyzer give a first guess; confirm it from the signals above. Record the front end too (`Server`, `Via`, `x-envoy-*`, CDN headers): the L3 differentials need both sides.

## Signing secrets and debug surfaces (any stack)

**L1 — Basic**
- [ ] Debug surfaces the framework ships with: Werkzeug debugger, Django `DEBUG = True`, Laravel Ignition/Whoops, Rails `/rails/info/*`, Symfony `/_profiler`, Spring actuator, ASP.NET detailed errors. Report the exposure. Never use a console, attempt a debugger PIN, or use credentials the page shows.
- [ ] Decode what your own A cookie carries: Flask sessions (`flask-unsign --decode --cookie '<A cookie>'`) and Express `cookie-session` are signed, not encrypted; JWTs are readable. Roles, IDs, and flags stored client-side go to `levers.md`.

**L2 — Intermediate**
- [ ] Default and weak keys, offline only. `badsecrets` checks a cookie, token, or ViewState issued to your own A against known default and published keys (Flask, Django, Rails, Laravel, Symfony, Express signed cookies, ASP.NET `machineKey`, JSF ViewState, JWT). For Flask, `flask-unsign --unsign --cookie '<A cookie>' --wordlist <list>`. Nothing is sent to the target.

**L3 — Advanced**
- [ ] Leaked keys (`SECRET_KEY`, `secret_key_base`, `APP_KEY`, `APP_SECRET`, `machineKey`, a session secret) from exposed files, repositories, or images: verify **offline** that the key validates the signature on your own A cookie or token. That is the liveness proof (`reporting.md` Gate 5); nothing is sent.
- [ ] Severity from the framework's documented behaviour, not from a test: session forgery everywhere; deserialization where the framework unserializes signed data (Rails cookie serializer `:marshal`/`:hybrid`, Laravel before 5.6.30, ASP.NET ViewState, JSF ViewState); signed `_fragment` rendering in Symfony. Report with the offline proof. Never forge a session or send a payload signed with the key.

## Server-rendered JS frameworks (Next.js, Nuxt, SvelteKit, Remix)

**L1 — Basic**
- [ ] **Data endpoints.** Each framework serves a page's server-side data as JSON at its own URL. Request it logged out, and as A for B's page:
  ```
  Next.js (pages router)   /_next/data/<buildId>/<page>.json        buildId from __NEXT_DATA__
  Nuxt                     /<page>/_payload.json                    prerendered / hybrid pages
  SvelteKit                /<page>/__data.json
  Remix                    /<page>?_data=routes%2F<route>           React Router 7 single fetch: /<page>.data
  ```
  Auth that lives in a layout, a client-side redirect, or a middleware `matcher` that does not cover the data URL leaves the data open.
- [ ] **Serialised state in the HTML:** `__NEXT_DATA__`, `__NUXT__`, RSC flight data (`self.__next_f`), SvelteKit's inline data. Look for fields the page never renders: other users' records, emails, internal IDs, server-only config (Nuxt private `runtimeConfig` keys copied into the public payload).

**L2 — Intermediate**
- [ ] **Next.js middleware as the only gate.** Check whether pages and route handlers enforce auth themselves or rely on middleware alone. CVE-2025-29927: the internal `x-middleware-subrequest` header makes self-hosted Next.js skip middleware — fixed in 12.3.5, 13.5.9, 14.2.25, and 15.2.3. The header value depends on the version (`middleware` or `src/middleware`, repeated five times with `:` on 13.2 and later — see the advisory). Prove on a page that redirects your logged-out browser to login.
- [ ] **Server actions and form actions are public POST endpoints**, whichever page imports them: Next.js `Next-Action` IDs, Remix/React Router `action` exports, SvelteKit `?/<name>` form actions. Collect action IDs from client chunks (including chunks of admin-only pages), then call each as A with B's IDs and as a lower role. Authorization has to be inside the action.
- [ ] SvelteKit form actions rely on the built-in `Origin` check (`csrf.checkOrigin`). Disabled, or forms posted to a `+server.js` endpoint that skips it → CSRF (`vectors-client.md`).
- [ ] **Next.js image optimiser** `/_next/image?url=…&w=64&q=75`: wildcard `remotePatterns`/`domains` make it fetch any host (external-only reach is usually N/A — `vectors-server.md` → SSRF); `dangerouslyAllowSVG` serves SVG from the app origin — check the CSP sent with it.
- [ ] `rewrites` to an external `destination` (visible as proxied responses): path traversal through the rewrite into the upstream's other routes.

**L3 — Advanced**
- [ ] **Matcher vs. router:** the middleware `matcher` is tested against one form of the path, the router serves another — trailing slash, locale prefixes (`/en/admin`), `/_next/data/…` forms, encoded characters.
- [ ] **RSC vs. HTML in the cache:** a request with `RSC: 1` returns the flight payload instead of HTML. A CDN that does not key on it can store the wrong representation at the page URL. Your own cache buster only; anything wider is approval-gated cache poisoning (`vectors-server.md`). Next.js pages-router cache poisoning, CVE-2024-46982 (13.5.1 to before 13.5.7, 14.0.0 to before 14.2.10, self-hosted), is in the same approval class.
- [ ] **React Server Components, CVE-2025-55182:** pre-auth RCE in RSC request decoding — `react-server-dom-webpack`/`-parcel`/`-turbopack` 19.0.0, 19.1.0, 19.1.1, 19.2.0 (fixed 19.0.1, 19.1.2, 19.2.1), shipped inside Next.js 15.x/16.x App Router and other RSC frameworks. Known CVEs rule above. Related: CVE-2025-55183 (server function source exposure).

## Spring (Boot, MVC, Security, Data REST)

**L1 — Basic**
- [ ] Actuator (`vectors-server.md` → Exposed files): `/actuator/env`, `/heapdump`, `/mappings`, `/configprops`, `/loggers`, `/gateway/routes` — also under a custom base path (`/manage`, `/admin/actuator`) or the management port. A heap dump is a finding by its existence and size; do not mine it for credentials to use.
- [ ] Error pages and stack traces that name classes and library versions — input for Known CVEs.

**L2 — Intermediate**
- [ ] **Security rules vs. MVC routing.** Spring Security decides on one form of the path; Spring MVC may route another. Against a URL that returns `403` for your role:
  ```
  /admin/          trailing-slash match: on by default before Spring Framework 6 / Boot 3
  /admin.json      suffix-pattern match: on by default before 5.3
  /ADMIN           when the app turns on case-insensitive matching
  /admin%0a        RegexRequestMatcher with "." — CVE-2022-22978, Spring Security before 5.5.7 / 5.6.4
  ```
  `StrictHttpFirewall` rejects `;`, `//`, `%2e`, `%2f`, and `\` by default. A `400` there is the firewall; it says nothing about apps whose authorization lives in a custom filter, an interceptor, or a gateway in front. For an anonymous `403`, the `access-bypass` first pass already sends the trailing-slash, `.json`, uppercase, and `%0a` forms; a `403` that depends on your role stays in Burp with A's session.
- [ ] **Spring Data REST:** the API root and `/profile` list auto-exported repositories — often entities the UI never shows. For each: list, `GET /<repo>/<id>`, `/<repo>/search`, `?projection=`, and `PATCH`/`PUT` with fields the UI never sends (`vectors-logic-api.md` → Mass assignment).
- [ ] Method security on one door only: `@PreAuthorize` on the REST controller but not on the GraphQL resolver, the Data REST repository, or the websocket `@MessageMapping` that reaches the same entity.

**L3 — Advanced**
- [ ] View-name injection: a controller that returns user input as the view name (`return "user/" + lang;`) runs Thymeleaf `__${…}__` preprocessing — arithmetic proof only (`vectors-injection.md` → SSTI, Expression language).
- [ ] Spring Cloud Gateway: a readable `/actuator/gateway/routes` is the finding (internal route map). Creating or refreshing a route writes to shared infrastructure (CVE-2022-22947) — operator approval, and only if the program allows it.
- [ ] Known CVEs by version: Spring4Shell (CVE-2022-22965: JDK 9+, WAR on Tomcat), Spring Cloud Function routing expressions (CVE-2022-22963).

## Rails

**L1 — Basic**
- [ ] **Format suffix:** append `.json` or `.xml` to HTML routes (`/projects/5.json`). `render json: @record` serialises every column — compare the fields with what the HTML shows.
- [ ] `/rails/info/routes`, `/rails/info/properties`, `/rails/mailers` outside development → route map and config.

**L2 — Intermediate**
- [ ] **GlobalID:** `gid://<app>/<Model>/<id>` values in forms, params, or Turbo Streams are resolved with `GlobalID::Locator.locate`, which loads whatever model the string names. Swap the model and the ID (`gid://app/User/<B>` where a `Project` was expected). `SignedGlobalID` adds a purpose and an expiry — check both are enforced.
- [ ] **Active Storage:** blob URLs (`/rails/active_storage/blobs/redirect/<signed_id>/<file>`) do not expire by default and are not tied to a user — a URL that leaks once is permanent access (Gate 2 covers the leak). Proxy-mode URLs and variants behave the same.
- [ ] Strong parameters skipped: `params.permit!`, `to_unsafe_h`, or a JSON endpoint that passes the parsed body to `update` — mass assignment with Rails naming (`user[role]=admin`, `vectors-logic-api.md`).
- [ ] Action Cable / Turbo Streams: stream names built from IDs — subscribe with B's (`vectors-logic-api.md` → Protocols).

**L3 — Advanced**
- [ ] `render` with user input (`render params[:page]`, `render inline:`) → file read or ERB evaluation; arithmetic proof only.
- [ ] Leaked `secret_key_base` → *Signing secrets* above.

## Laravel

**L1 — Basic**
- [ ] Debug mode: a `500` that renders Ignition or Whoops with environment variables (`APP_DEBUG=true`).
- [ ] `/.env`, `/storage/logs/laravel.log`, `/telescope`, `/horizon`, `/_debugbar/open` (Laravel Debugbar keeps earlier requests, other users' included — open at most one and redact).

**L2 — Intermediate**
- [ ] **Livewire public properties** are client state sent back on every update (`wire:snapshot`; `/livewire/update` in v3). Change a property that carries an ID or a role (`userId`, `teamId`, `isAdmin`) in the request's `updates`. Livewire v3 refuses client changes only to properties marked `#[Locked]`; authorization has to be in the action.
- [ ] Mass assignment: models with `$guarded = []` or a wide `$fillable` — send the extra fields (`vectors-logic-api.md`).
- [ ] **Signed URLs** (`URL::signedRoute`, `temporarySignedRoute`): change, add, or drop query parameters; swap the path ID; replay after `expires`. Routes checked with `hasValidSignatureWhileIgnoring([...])` let the ignored parameters change freely.
- [ ] Policies on one door only: the controller checks a Policy, but the API route, the Nova/Filament admin resource, or the Livewire component reaching the same model does not.

**L3 — Advanced**
- [ ] Leaked `APP_KEY` → *Signing secrets* above.
- [ ] Known CVEs: Ignition `/_ignition/execute-solution` with debug mode on (CVE-2021-3129, Ignition before 2.5.2); Livewire v3 hydration (CVE-2025-54068, fixed in 3.6.4).

## Django and Django REST Framework

**L1 — Basic**
- [ ] `DEBUG = True`: a 404 lists every URL pattern; a 500 shows settings (partly masked) and local variables.
- [ ] DRF browsable API (`?format=api`) and `?format=json` on views that render HTML.

**L2 — Intermediate**
- [ ] **Filters on hidden fields.** `OrderingFilter` with `ordering_fields = '__all__'` sorts on any model field, including ones the serializer hides (`?ordering=password`, `?ordering=-reset_token`); `DjangoFilterBackend` with `filterset_fields = '__all__'` filters on them. Either is an oracle: prove by sorting or filtering on a hidden field of your own A and B rows, one character (`vectors-injection.md` → ORM leaks).
- [ ] **Viewset scoping:** `get_queryset` restricted to the user, but an `@action`, a nested router, or a second viewset reaches the same model through `Model.objects.all()`. As A, try every list, detail, and extra action on B's object.
- [ ] Views that pass the query string to the ORM (`filter(**request.GET.dict())`) accept any lookup the client names (`owner__email__startswith=`) — ORM leak.

**L3 — Advanced**
- [ ] Version-specific ORM SQL injection where user-controlled keys reach the affected API: `_connector` in `Q()`/`filter(**kwargs)` (CVE-2025-64459), JSONField keys in `values()`/`values_list()` (CVE-2024-42005), `Trunc`/`Extract` `kind` (CVE-2022-34265). Check the version and whether the view passes client keys through; one time-based probe as proof.
- [ ] Leaked `SECRET_KEY` → *Signing secrets* above.

## Express and NestJS

**L1 — Basic**
- [ ] **Routing defaults:** Express routes are case-insensitive and ignore a trailing slash unless the app sets `caseSensitive`/`strict`. Middleware that checks `req.path === '/admin'` or `startsWith('/admin')` lets `/Admin` and `/admin/` through to the router — try both on every `403` (the `access-bypass` first pass covers the trailing slash and an uppercased path for anonymous blocks).
- [ ] `qs` turns `a[b]=c` and `a[]=1` into objects and arrays in query strings and form bodies → NoSQLi and type confusion (`vectors-injection.md`).

**L2 — Intermediate**
- [ ] NestJS `ValidationPipe` without `whitelist: true` (or `forbidNonWhitelisted`) passes undeclared properties to the ORM — mass assignment; nested DTOs without `@ValidateNested()` and `@Type()` are not validated at all.
- [ ] `trust proxy` set too wide: `req.ip` comes from a client-supplied `X-Forwarded-For`, so IP allowlists and per-IP limits keyed on it move with the header.
- [ ] Guards on one door only: the REST controller has the guard; the GraphQL resolver, the microservice `@MessagePattern` handler, or the websocket gateway reaching the same service does not.

**L3 — Advanced**
- [ ] **Request objects as render locals** (`res.render(view, req.query)` or `req.body`): the client can set template-engine options as well as variables, and several engines have advisories where an option reaches code generation (EJS CVE-2022-29078 and later). Prove only that a request-supplied local overrides a template variable, note the engine and version, and stop (`vectors-injection.md` → SSTI).
- [ ] Server-side prototype pollution through merge helpers: `vectors-injection.md`.

## ASP.NET, IIS, and Blazor

**L1 — Basic**
- [ ] Detailed errors (`customErrors Off`, the developer exception page), `/trace.axd`, `/elmah.axd` (`vectors-server.md`).
- [ ] IIS short names (`~1`, `shortscan`): informational alone; feed the recovered prefixes into content discovery.

**L2 — Intermediate**
- [ ] **Cookieless session segments:** `/(S(x))/` and `/(A(x))/` inside the path — path-based rules can be evaluated on a different form of the path than the one served (CVE-2023-36899, CVE-2023-36560, patched in 2023). Try them by hand against a `403`ed path — they are not in the `access-bypass` set.
- [ ] Model-binding over-posting: actions that bind a whole entity without `[Bind]` or a view model accept every property — mass assignment (`vectors-logic-api.md`).
- [ ] **Blazor Server:** UI events and JS interop travel over the `/_blazor` SignalR connection. Any static `[JSInvokable]` .NET method can be called from the browser console with arguments of your choosing (`DotNet.invokeMethodAsync('<Assembly>', '<Method>', …)`) — authorization has to be inside the method.

**L3 — Advanced**
- [ ] Request filtering and `<location>` authorization vs. routing: `%2F`, `\`, trailing `.` or space, `::$DATA`.
- [ ] ViewState with MAC disabled (`vectors-injection.md` → Deserialization, DNS-only detection) or a known or leaked `machineKey` (*Signing secrets* above).

## WordPress, Drupal, Symfony, and other PHP
WordPress bugs that pay are mostly in plugins, not core; Drupal ones are mostly in what modules expose through their APIs. PHP type juggling is in `vectors-logic-api.md` → Type confusion.

**L1 — Basic**
- [ ] Inventory into `map.md`: core version (generator tag, `?ver=`), plugins and themes (`/wp-content/plugins/<slug>/readme.txt` → `Stable tag`, REST namespaces in `/wp-json/`). `wpscan --url https://<host> --headers "$HDR" --throttle <ms from RL> --enumerate vp,vt --plugins-detection passive` where automation is allowed.
- [ ] User listing at `/wp-json/wp/v2/users` or `/?author=1` is informational on nearly every program; the usernames go to `levers.md`. `xmlrpc.php` enabled is N/A on its own.
- [ ] Drupal: JSON:API and REST exposure — `/jsonapi`, `/jsonapi/user/user`, `/node/<id>?_format=json` — for fields the HTML hides, unpublished content, and user emails.

**L2 — Intermediate**
- [ ] **AJAX actions** (`/wp-admin/admin-ajax.php?action=<name>`): handlers registered with `wp_ajax_nopriv_<name>` run for anyone, and `wp_ajax_<name>` for any logged-in user, subscribers included. Each must check `current_user_can()` itself. Find action names in plugin JS and in the plugin's source — most plugins are public on wordpress.org, so read the code for the installed version. Test every action anonymously and as a subscriber (register one only if registration is open and the program's account rules allow it).
- [ ] **REST routes** under `/wp-json/<namespace>/`: a route registered with `'permission_callback' => '__return_true'`, or one that checks only for a logged-in user, is open to subscribers. `?rest_route=/<namespace>/<route>` reaches the same handler when the edge blocks `/wp-json/`.
- [ ] Nonces are CSRF tokens, not authorization: a nonce printed on a page subscribers see does not protect an admin action whose handler skips the capability check.
- [ ] Symfony `access_control` rules are regexes on the raw path; compare them with the routes they protect — a `^/admin` rule does not cover a locale-prefixed `/en/admin`, and an anchored rule can miss trailing-slash or `_format` variants.

**L3 — Advanced**
- [ ] Known plugin and module CVEs: installed version inside the advisory's range (WPScan, Wordfence Intelligence, Patchstack, Drupal security advisories), then a non-destructive detection, per the Known CVEs rule at the top of this file.
- [ ] Application passwords and plugin API keys that keep working after the user's role changes or the user is removed.
- [ ] Generic PHP: source disclosure from editor backups (`.php~`, `.php.bak`, `.php.swp`) served as text; `phpinfo()` pages; `php://filter/convert.base64-encode/resource=` in an LFI parameter turns an include into a source read — one file as proof, secrets redacted (`vectors-server.md` → Path traversal).

## Flask and FastAPI

**L1 — Basic**
- [ ] Flask: Werkzeug debugger and tracebacks (*Signing secrets and debug surfaces* above: report the exposure, never the console or the PIN); session contents decoded from your own A cookie.
- [ ] FastAPI: `/docs`, `/redoc`, `/openapi.json` give the full operation list with request models; operations the UI never calls go to the tracker. Validation errors name the fields and types of any body you send (`loc`, `type`) → `levers.md`.

**L2 — Intermediate**
- [ ] **Where auth is attached.** Flask: per blueprint (`before_request`) or per view (`@login_required`). A view registered on the app or another blueprint skips the blueprint's hook, and `@login_required` written above `@app.route` is never applied. FastAPI: `Depends()` per route or per router; routes defined on `app` directly, or on a router included without the `dependencies=` the others use, skip it. Black-box: run every operation in `openapi.json` (or every route seen) anonymously and as A, and compare siblings.
- [ ] **Model binding** (FastAPI/Pydantic): the request model names the fields the handler accepts. Send the response model's extra fields (`is_admin`, `owner_id`, `org_id`) in requests anyway — handlers that build the ORM object from `**body` or a shared model accept them. A handler that returns the database model without a narrower `response_model` leaks its fields: check responses for `hashed_password` and internal flags.
- [ ] Flask file helpers (`send_file`, `send_from_directory`) with a path from the request → `vectors-server.md` → Path traversal. `render_template_string` with user input → `vectors-injection.md` → SSTI (Jinja2).

**L3 — Advanced**
- [ ] Leaked `SECRET_KEY` → *Signing secrets* above, offline only.
- [ ] Proxy vs. app paths: Werkzeug and Starlette route on the decoded path; a proxy rule on the raw path treats `%2f`, `//`, and trailing-slash redirects differently (*Front-end servers and proxies* below).

## Go services (net/http, Gin, Echo, Fiber, Chi)

**L1 — Basic**
- [ ] `/debug/pprof/` and `/debug/vars` (`net/http/pprof` or `expvar` on a public listener): the index alone proves the exposure. `cmdline` and `expvar` output can carry flags and secrets — open at most one and redact. Heap and goroutine dumps are not needed for the report.

**L2 — Intermediate**
- [ ] **Route groups:** Gin, Echo, Chi, and Fiber attach auth middleware to a group (`r.Group("/admin", auth)`); a handler registered on the root router or another group by mistake has none. Compare every route under a prefix anonymously and as A.
- [ ] **JSON binding** (`encoding/json`): keys match struct fields case-insensitively, and the last of duplicate keys wins. When a gateway or validator in another language checks the body and the Go service binds it, `{"role":"user","Role":"admin"}` reads as `user` to one and `admin` to the other (Bishop Fox, *An Exploration of JSON Interoperability Vulnerabilities*, 2021). Unknown fields are dropped unless the service calls `DisallowUnknownFields`, so mass assignment needs a field the struct really has — take names from responses and errors.
- [ ] Path handling: `net/http`'s `ServeMux` cleans `..` and `//` and answers with a redirect; Gin's `RedirectTrailingSlash` and `RedirectFixedPath` redirect slash and case variants. A middleware or proxy that matched the raw path before the redirect sees a different path from the handler after it.

**L3 — Advanced**
- [ ] `text/template` where `html/template` belongs: no contextual escaping → XSS in rendered pages. Template input is data, not code, so this is XSS, not SSTI, unless the template text itself comes from the user.
- [ ] gRPC-Gateway and Connect services: the REST mapping and the gRPC method reach the same handler through different interceptors (`vectors-logic-api.md` → gRPC).

## Phoenix (Elixir)

**L1 — Basic**
- [ ] Dev surfaces in production: the `Plug.Debugger` error page (code excerpts and bindings), `/dev/dashboard` (LiveDashboard), `/phoenix/live_reload/socket`.
- [ ] Decode your own A cookie: a value starting `SFMyNTY.` is signed but not encrypted — the middle part is a Base64url Erlang term, readable in `iex` with `:erlang.binary_to_term(Base.url_decode64!(part, padding: false), [:safe])` → roles and IDs to `levers.md`. A value starting `QTEyOEdDTQ` is encrypted.

**L2 — Intermediate**
- [ ] **LiveView events:** every `phx-click`, `phx-submit`, and `phx-change` event reaches `handle_event/3` over the websocket with parameters the client chooses (`phx-value-*`). Authorization checked in `mount/3` does not cover them: change an ID in an event payload to B's object. Same model as Laravel Livewire above.
- [ ] **Channels:** `join/3` decides who may join a topic (`room:<id>`, `user:<id>`) — join B's topic as A. `handle_in/3` must re-check per message; an authorized join does not authorize every message.

**L3 — Advanced**
- [ ] Leaked `secret_key_base` → *Signing secrets* above, offline only.
- [ ] `Phoenix.Token`: tokens verified with `max_age: :infinity`, or a token minted for one purpose (salt) accepted where another is expected — a password-reset token used as a socket token.

## Front-end servers and proxies (nginx, Apache, Envoy)

**L1 — Basic**
- [ ] Fingerprint the front end per host — `Server`, the shape of its own error pages, `Via`, `x-envoy-*`, CDN headers — and note where it ends and the app begins (a `404` in the proxy's style vs. the app's).

**L2 — Intermediate**
- [ ] **Identity headers between hops.** Headers a gateway or auth proxy is meant to set — `X-User-Id`, `X-Forwarded-User`, `X-Remote-User`, `X-Auth-Request-Email` (oauth2-proxy), `X-Tenant-Id`, `X-Original-URL`, `X-Rewrite-URL` — send them yourself. With A's session, set the identity header to B: does the back end act as B? A back end that trusts them is safe only if every proxy in front strips them. `access-bypass` sends only the IP-spoof and URL-rewrite headers; user-ID and email headers are manual.
- [ ] Proxy rules vs. app routing: `location /admin { deny all; }` against `/%61dmin`, `/admin%2f`, `/ADMIN`, `/admin/`, `/admin;x` — whichever side decodes or normalises more wins. Run the `access-bypass` first pass on the blocked URL, then send by hand what it does not: single-character encodings (`/%61dmin`), `%2f` suffixes, rules that need a session.
- [ ] nginx: `$uri` is decoded, so `$uri` in `return`, `add_header`, or `proxy_pass` allows CRLF via `%0d%0a` in the path (`vectors-injection.md` → CRLF); `proxy_pass` with a URI part normalises the path before forwarding, without one it forwards the raw path; `alias` off-by-slash (`vectors-server.md` → Path traversal).
- [ ] Envoy / Istio: path normalisation settings (`normalize_path`, `merge_slashes`, `path_with_escaped_slashes_action`) against `AuthorizationPolicy` path rules; `#` and `?` inside the path (Istio CVE-2021-39156).

**L3 — Advanced**
- [ ] **Apache confusion attacks** (Orange Tsai, Black Hat USA 2024; fixed in 2.4.60): `mod_rewrite` results are treated as both URL and filesystem path. An encoded `%3F` truncates the rewritten path (ACL bypass, CVE-2024-38474 — e.g. `/admin.php%3Fx.php` against an IP-restricted `admin.php` behind PHP-FPM); rewrites can resolve against the filesystem root as well as the document root (source disclosure, CVE-2024-38475); backend response headers can switch the handler (CVE-2024-38476). Check the version first; test only against rules you can see blocking you. Not in the `access-bypass` set.
- [ ] Two proxies in a row (CDN → nginx → app): each normalisation layer is a differential. Map which layer applies each rule before testing.

## Auth platforms (Firebase, Cognito, Auth0, Keycloak, Auth.js)

These calls go to the platform's own hosts, acting on the target's tenant or project. Confirm the program counts that tenant as in scope (operator queue if the scope table is silent), and that its test-account rules allow accounts created this way.

**L1 — Basic**
- [ ] **Self-signup the UI hides.** The client config is public by design; the platform's signup API may still be on when the app is invite-only:
  ```
  Firebase   POST https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=<apiKey>
  Cognito    aws cognito-idp sign-up --client-id <ClientId> --username <alias> --password <pw> --region <region>
  Auth0      POST https://<tenant-domain>/dbconnections/signup   {"client_id":…,"connection":"<db connection>","email":…,"password":…}
  Keycloak   /realms/<realm>/protocol/openid-connect/registrations?client_id=<id>&response_type=code&scope=openid
  ```
  The account is not the finding; what it reaches is — database rules of the form `request.auth != null`, APIs that accept any token from the pool, data scoped by an unverified email claim. Aliases only; a Cleanup ledger row per account.

**L2 — Intermediate**
- [ ] **User-writable attributes.** Cognito: `aws cognito-idp update-user-attributes --access-token <A's>` on `custom:*` attributes (`custom:role`, `custom:tenant_id`) and on `email` — the app client's write permissions decide. A changed `email` is unverified, but an app that keys on the `email` claim without checking `email_verified` logs A in as B. Keycloak: attributes A can edit in the account console that a protocol mapper copies into tokens. Auth0: `user_metadata` (user-editable) read as if it were `app_metadata`.
- [ ] Tokens from the same pool or tenant issued to another app client (mobile, a sister product) accepted here because `aud`/`client_id` is not checked (`vectors-auth.md` → JWT, cross-service).
- [ ] Auth.js / NextAuth: `allowDangerousEmailAccountLinking` on a provider that does not verify emails (account linking by email); `callbackUrl` validation on sign-in and sign-out (`vectors-client.md` → Open redirect).

**L3 — Advanced**
- [ ] Database and storage rules: Firestore `get` vs. `list` asymmetry, a rule on a parent path that a sibling path lacks, Storage rules that check `request.auth` but not the path's user ID. One record, your own B's. Cognito identity pools: `vectors-server.md` → Cloud storage.
- [ ] Federation: the platform links an external IdP identity to an existing local user by email (`vectors-auth.md` → OAuth, nOAuth pattern).

## Enterprise SaaS platforms (Salesforce Experience Cloud, ServiceNow)

Test only an instance the scope table lists (the program's custom domain, `*.my.site.com`, or `<instance>.service-now.com`), never the vendor platform itself. Findings here are the program's configuration, not the vendor's bugs. Proof: one record, redacted, preferably your own B's.

**L1 — Basic**
- [ ] Salesforce: as the guest user, `POST /s/sfsites/aura` (also `/aura`, `/sfsites/aura`) with the `aura.context` from a page load. Request the list of objects the site exposes, then one record of each custom object (`__c`) at the smallest page size. Guest-readable objects holding customer data are the finding.
- [ ] ServiceNow: knowledge-base and Service Portal endpoints that answer without a session (`/kb_view.do`, `/api/now/sp/…` widgets) — check which records come back to an anonymous session.

**L2 — Intermediate**
- [ ] Salesforce, as A (a community user): B's records through record and list actions — sharing rules vs. object permissions; `@AuraEnabled` Apex methods called directly with other IDs.
- [ ] ServiceNow, as A: the table API (`/api/now/table/<table>`) on tables the portal never shows; `sysparm_query` filters on fields A cannot read (an oracle, one character).

**L3 — Advanced**
- [ ] SOQL injection in custom Apex methods that build queries from parameters — boolean pair only (`vectors-injection.md`).
- [ ] ServiceNow template injection in Jelly/UI macros (CVE-2024-4879 class): instance version and patch level first; non-destructive detection only.
