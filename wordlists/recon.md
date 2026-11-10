# Recon

Everything is scope-filtered before it reaches an active tool. Set `RL` to the program's rate limit (requests/sec) and `HDR` to the required research header.

```bash
S=$PWD/.claude/skills/h1-app-test; P=$PWD/engagements/<program>   # as in P0 step 1: set from the repository root, so both stay valid after the cd
D=target.com; RL=5                             # scope.md → Rules; 5 when the policy states no limit (P0 step 4)
HDR='X-HackerOne-Research: <h1user>'
mkdir -p $P/recon && cd $P/recon
```

Run the tiers in order. L1 gives a working map of the obvious surface. L2 finds the surface the UI hides. L3 finds the surface other hunters missed — on a mature wildcard program, that is where untested hosts live.

### 0. Tool check
The commands below were written against specific releases, and these tools rename and drop flags between versions. Before the first run of an engagement:
```bash
for t in subfinder amass dnsx httpx katana gau waybackurls ffuf nuclei tlsx asnmap alterx puredns massdns jsluice trufflehog arjun x8 interactsh-client; do
  printf '%-18s %s\n' "$t" "$(command -v "$t" || echo MISSING)"
done | tee tools.txt
```
- Add each installed tool's version to `tools.txt` (`<tool> -version`; `gau --version`; `ffuf -V`) and compare every flag this file uses with `<tool> -h`. A flag that is gone gets fixed in the command, and noted in the P1 handoff.
- Checked in October 2026 against subfinder 2.14.0, httpx 1.9.0, katana 1.6.1, nuclei 3.8.0, ffuf 2.1.0, asnmap 1.1.1, amass 5.1.1, trufflehog 3.84.2, dnsx 1.3.1, tlsx 1.4.0, alterx 0.1.0, and puredns 2.1.1: every flag in §1–§5, §10, and §11 is present except as listed here.
  - **amass 5:** passive is the default and `-passive` is deprecated; `enum` has no `-o` and writes to its own database, so §1 reads the names back with `amass subs -names`. Kali's `/usr/bin/amass` wrapper runs `sudo libpostal_data download` on first use; run that once by hand (or call `/usr/lib/amass/amass`) so it does not stall a script.
  - **httpx:** two tools share the name. ProjectDiscovery's prints a `projectdiscovery.io` banner on `httpx -version`; the Python `httpx` client's CLI rejects these flags.
  - **trufflehog 3:** `--no-verification` is a global flag, listed in `trufflehog --help`, not under `trufflehog filesystem -h`.
  - **puredns 2:** `-l`/`--rate-limit` is unlimited by default, and every brute-force name is a cache miss that the target's authoritative DNS answers, so §10 sets it. It needs `massdns` on `PATH` (or `-b <path>`), and reads `~/.config/puredns/resolvers.txt` when `-r` is not given — if that file does not exist the run dies with no resolvers, so seed it before the first use. A handful of public resolvers is enough to make the tool work; a real brute force wants a vetted, de-duplicated list, because a poisoned or rate-limited resolver silently costs you names.
  - **massdns** has no package on some boxes. `git clone --depth 1 https://github.com/blechschmidt/massdns && make` builds it with no dependencies and no root; symlink `bin/massdns` onto `PATH`.
  - **x8 4.3.0:** `--learn-requests` defaults to 9 calibration requests per URL, which is what makes its results trustworthy on a page that varies; `--verify` and `--strict` cut the rest. `-d` is milliseconds between requests — set it from `$RL` like every other tool here. Install from the project's release binary (`x86_64-linux-x8.gz`); it is a static Rust binary with no runtime to break.
  - **xnLinkFinder 8.2:** `-sf`/`--scope-filter` is **mandatory** (it was optional in earlier versions) — a domain or a file of domains, no scheme. Without it the tool prints a message and exits, including for offline directory input. Install with `pipx install xnLinkFinder`.
  - **unwebpack-sourcemap** is a single script with no package. Clone it, give it its own venv with *current* `requests` and `beautifulsoup4` — its `requirements.txt` pins 2019 releases that will not build on a modern Python — and shim it onto `PATH`.

A missing tool gets its fallback, and the handoff says what was lost:

| Missing | Fallback | What is lost |
|---|---|---|
| subfinder, amass | The crt.sh query in §1 | Passive sources that need API keys |
| dnsx | `dig +short CNAME <host>` per host, paced at `RL`; `httpx` resolves while it probes | The bulk CNAME list for takeover triage |
| httpx | `curl -s -o /dev/null -w '%{http_code} %{url_effective}\n' -H "$HDR" https://<host>/` per host, paced at `RL`; Wappalyzer in the browser for the stack | Tech detection, titles, CDN flags |
| katana | gau and waybackurls, plus the proxy history from the P2 walk | Endpoints parsed from live bundles; §6 by hand recovers part of it |
| ffuf | Skip §4's brute force; request the high-value path list with curl | Wordlist coverage |
| nuclei | Skip §5; the high-value paths and `vectors-frameworks.md` → Fingerprints by hand | Template breadth |
| jsluice, xnLinkFinder | `grep -Eho '"/api/[^"]*"' js/*.js` and the same for `/v[0-9]+/` | Paths built by string concatenation — and the grep assumes a path shape, which is how `/api/awin/order` was missed on a real target while sitting in the corpus |
| x8 | `arjun -u <url> --rate-limit $RL`, or `ffuf` with a size filter set from a measured baseline | Calibrated page comparison; without it you must measure the page's own variance by hand before trusting any diff |
| trufflehog | `jsluice secrets js/*.js`, or grep for key prefixes (`AKIA`, `ghp_`, `xoxb-`, `sk_live_`) | Detector coverage |
| tlsx, asnmap, alterx, puredns | Skip that L3 step | The L3 surface it would have found |
| interactsh-client | Your own interactsh server (`interactsh-client -server <yours> -token "$(cat $H1_HOME/interactsh-token)"`), Burp Collaborator, or `scripts/oob_listener.py serve` on a VPS you control (HTTP only; `token` mints a correlation token, `poll --save` copies the hit into a finding's `evidence/`) | DNS-only callbacks (JNDI `dns://`, XXE with no HTTP egress) unless you also run a DNS listener |

A skipped or degraded step leaves its recon row `partial` in the tracker, naming the tool, and goes in the P1 handoff under gaps.

---

## L1 — Basic

### 0b. Where these commands run
**Anything that takes minutes runs on the VPS, not on this box** — §1–§5, §10, §12, §13,
and any bundle pull larger than a handful of files. Three reasons, in order of how often
they bite:

1. **The source address is the engagement's bound exit.** `egress.py` binds both roles to
   the VPS (P0 step 4), so running the tool *there* means recon traffic leaves from the
   same address the testing traffic does, and from the address the report's *Scope of
   testing* names. Running it locally through the SOCKS tunnel sends the same packets but
   adds a hop that drops under load, and running it locally *without* the tunnel spends
   the home address — the one thing that cannot be replaced.
2. **A local restart kills a local job.** A workstation reboot, a WSL restart or a closed
   laptop ends a two-hour brute force and leaves a half-written output file. On the VPS
   under `tmux` it survives all three.
3. **The home address stays in reserve for testing.** Keeping recon off it is what keeps
   it clean enough to be the fallback when a proxied exit is refused.

```bash
ssh vps 'mkdir -p ~/recon/<program>'
scp $P/scope.json $P/out-of-scope.regex vps:~/recon/<program>/     # the gate goes too
ssh vps "tmux new -d -s recon-<program> 'cd ~/recon/<program> && <the command> |& tee <tool>.log'"
ssh vps 'tmux ls'                                                  # still running?
rsync -av vps:~/recon/<program>/ $P/recon/                         # results come back
```

`tmux`, not `nohup`: a detached session can be reattached to watch progress, and a
dropped ssh connection does not take the job with it.

**Tool parity is a precondition, not an assumption — and parity means versions, not just
presence.** The VPS needs the same binaries this box has, or the command silently becomes
a different command. Worse, a freshly provisioned VPS usually has *newer* releases than a
workstation installed months ago (measured 2026-10-25: subfinder 2.17.0 / katana 1.8.0 /
nuclei 3.11.1 there against 2.14.0 / 1.6.1 / 3.8.0 here), and these tools drop and rename
flags between minor versions. Record both hosts' versions in `tools.txt`, run §0's flag
comparison against **the host that will execute the command**, and note any divergence in
the P1 handoff. Check before the first run, and treat a missing tool there exactly as
§0's fallback table says:

```bash
ssh vps 'for t in subfinder amass dnsx httpx katana gau waybackurls ffuf nuclei tlsx \
  asnmap alterx puredns massdns jsluice trufflehog xnLinkFinder sourcemapper; do \
  printf "%-18s %s\n" "$t" "$(command -v $t || echo MISSING)"; done'
```

**What stays local:** the browser and its proxy history, Burp, the phone on USB, anything
needing a GUI, and the short targeted probes of P3/P4 where one request answers one
question. The split is duration and address, not capability.

### 0c. Information gathering — the coverage list
Fourteen questions that must each end in an answer, not a blank. Most are answered by the
sections below; the five marked **here** have no other home and are the ones that get
skipped, which is why they are written out. Record every answer in `map.md` →
*Information gathering*; an item that genuinely does not apply is `n-a` **with its
reason**, never silence.

| # | Item | Answered by |
|---|---|---|
| 1 | Manually explore the site | P2's walk, §8b |
| 2 | Spider/crawl for missed or hidden content | §3 (katana), §4 |
| 3 | Files that expose content: `robots.txt`, `sitemap.xml`, `.DS_Store` | §4, and **here** |
| 4 | Caches of major search engines | **here** |
| 5 | Content differences by User-Agent (mobile site, search-engine crawler) | **here** |
| 6 | Web application fingerprinting | §2 |
| 7 | Technologies used | §2, `vectors-frameworks.md` |
| 8 | User roles | P2's walk → `map.md` |
| 9 | Application entry points | §3, §6, §8 |
| 10 | Client-side code | §6 |
| 11 | Multiple versions/channels: web, mobile web, mobile app, web services | **here** |
| 12 | Co-hosted and related applications | §11, and **here** |
| 13 | Debug parameters | **here** |
| 14 | Published documentation and machine-readable specs | §8, `references/docs-mining.md` |

**3 — read the files, do not status-check them.** A `200` on `sitemap.xml` is not an
answer; the URLs inside it are. It is the cheapest entry-point enumeration there is, and
it partly covers for a crawl you could not run.
```bash
curl -s -H "$HDR" "https://$H/robots.txt" | tee robots.txt        # Disallow lines are a map
curl -s -H "$HDR" "https://$H/sitemap.xml" | grep -oE '<loc>[^<]+' | cut -c6- | tee sitemap-urls.txt
curl -s -H "$HDR" "https://$H/sitemap_index.xml" | grep -oE '<loc>[^<]+' | cut -c6-
for f in .DS_Store .git/config .env .well-known/security.txt; do
  printf '%-28s %s\n' "$f" "$(curl -s -o /dev/null -w '%{http_code} %{size_download}' -H "$HDR" "https://$H/$f")"
done
```
Feed `sitemap-urls.txt` into `urls.all.txt` (§3) before collapsing to shapes.

**Archived `robots.txt` is a better file than the live one**, because its `Disallow` lines
name the paths somebody wanted hidden at the time — including ones since removed from the
live copy, which is a stronger signal than their still being listed. Pull it per host
across dates from the Wayback Machine beside the URL corpus, and **replay what it names,
live**. An old endpoint being delisted is not the same as its being switched off, and the
ones worth trying first are the auth-shaped paths: a legacy sign-in, token-exchange or
impersonation endpoint left routed will still issue a session, which makes it a finding on
its own rather than a lead. Anything that answers goes into `urls.all.txt` and gets a
tracker row like any other surface — and into `surfaces.md`'s **era** axis, because a
legacy path that is still live is a parallel implementation by definition.

**4 — search-engine caches and indexes.** What the crawler was served is often not what
you are served: older builds, staging hosts, parameters nobody links to any more. Run the
`site:`/`inurl:` queries **twice, once with a mobile User-Agent**: search engines serve
different result sets to mobile clients, and a mobile-only host or path is a separate
implementation (`surfaces.md` → channel). Use `-keyword` exclusions to strip results you
have already processed, and expand "omitted similar results" rather than trusting the first
page. `site:` and
`inurl:` queries against the in-scope apexes, plus the Wayback corpus §3 already
pulls. Read-only, no traffic to the target. Anything found goes through `scope_guard.py`
like any other host before it is requested.

**5 — content differences by User-Agent.** Two UAs matter beyond the browser string: a
**mobile** UA and a **search-engine crawler** UA. Sites that geo-redirect or gate on
device routinely exempt crawlers, so a host that sends *you* off-scope may serve the real
application to Googlebot — which reopens a branch rather than evading a control.
```bash
for ua in "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/141.0.0.0 Safari/537.36" \
          "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 Mobile/15E148" \
          "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
          "curl/8.5.0"; do
  printf '%-14s %s\n' "${ua:0:12}" \
    "$(curl -s -o /dev/null -w '%{http_code} -> %{redirect_url} (%{size_download}B)' -H "$HDR" -A "$ua" "https://$H/")"
done
```
A status, a redirect target or a body size that moves with the UA is a finding-shaped
observation: record all four responses in `map.md` → *Anomalies*. **Then do not wear the
costume** — the point is that the content differs, not to test as a crawler. If a crawler
UA opens a host you could not otherwise reach, that is an operator-queue item before any
further request.

**11 — channels are separate surfaces, not one surface.** Enumerate them and say which you
tested: desktop web · mobile web (`m.`, `touch.`, `mobile.`) · the mobile apps and their
API (often a different `app-id`/tenant and a different auth path) · public web services ·
partner or admin portals. Two channels calling one backend disagree about what the client
enforces, and the weaker one wins. Regional tenants count: a per-region `app-id` means
per-region sessions and per-region data.

**13 — debug parameters.** Use `x8`, which calibrates before it tests:

```bash
x8 -u 'https://$H/me/login' -w params-debug.txt -X GET POST \
   --learn-requests 12 --verify --strict -d 200 --mimic-browser -H "$HDR"
```

| flag | why it is not optional |
|---|---|
| `--learn-requests` (default 9) | sends N calibration requests **first** and learns what the page varies by on its own. This is the whole ballgame — see the note below |
| `--verify` | re-checks every hit before reporting it |
| `--strict` | reports only parameters that changed a *different part* of the page, not its noisy part |
| `-d 200` | pacing; keep it at or under `$RL` |

**Calibrate, or your results are the page breathing.** Measured 2026-10-25 on a live
storefront: five identical requests to a product-listing page, no parameter at all,
returned 771 099 / 788 421 / 797 433 / 784 746 / 789 978 bytes — a **26 334 B spread**. A
hand-rolled size diff flagged 19 of 24 parameter names on that page; all 19 were the
product grid changing. The same sweep against a login page whose five-sample spread was
**246 B** flagged **0 of 50**. Pick a stable page, measure its spread before you start,
and set the threshold from the measurement — or let `--learn-requests` do it.

Wordlist: the names below, plus everything harvested from the bundles in §6. Keep both in
`params-debug.txt`.

```
debug  _debug  __debug  debug_mode  isDebug  debugbar  profiler  trace  verbose
test  dev  dev_mode  staging  beta  sandbox  mock  draft  preview  internal  admin
env=dev  env=staging  env=pre  env=sandbox  environment=dev
show_errors  showErrors  display_errors  error_reporting=E_ALL  phpinfo
XDEBUG_SESSION  XDEBUG_SESSION_START  profile  oseid  preview_theme_id  ls
```
Values to pair with each: `1`, `true`, `on`. The framework-specific half of that list
(`XDEBUG_*`, `debugbar`, `display_errors`, `error_reporting`, `phpinfo`) comes from
**debugHunter** (`github.com/devploit/debugHunter`), which also ships the exposed-path and
header lists §0c item 3 and the host-header row use.

**The extension itself is not run during a scoped engagement**, and the reason is
structural rather than a judgement about the tool: its only scoping control is a
*Whitelist* that means "domains to skip", so there is no way to say *only these hosts*.
With `host_permissions: <all_urls>` it probes every site the browser visits, which
violates hard gate 1 and sends unheadered probes to identity providers, analytics and
CDNs. Take its wordlists; leave the extension.

`arjun -u <url> -m GET|POST|JSON --rate-limit $RL` remains the quick pass for parameters
the application admits to.

**14 — the documentation is surface, and it is the only surface written by the people who
built the thing.** Two answers are required, and they are different questions. *The
contract:* is there an OpenAPI/Swagger, Postman, WSDL, `$metadata` or GraphQL schema, and
has `api_contract.py` imported it — a spec found and not imported counts as not found.
*The claims:* what does the vendor state in writing about who may read or change what, and
does the implementation honour each sentence. The second is passive, needs no ruling and no
requests, and is where the cheapest high-confidence rows on a mature program come from: the
quote is the impact statement, so a disagreement is a finding that writes itself. Both land
in `docs-claims.md`; the method is `references/docs-mining.md` and it is required reading
before §8.

### 1. Passive subdomain enumeration
```bash
subfinder -d $D -all -silent -o subs.subfinder.txt
amass enum -d $D                                  # amass 5: passive by default, results go to its own database
amass subs -names -d $D > subs.amass.txt
curl -s "https://crt.sh/?q=%25.$D&output=json" | jq -r '.[].name_value' | tr 'A-Z' 'a-z' | sed 's/^\*\.//' | sort -u > subs.crtsh.txt
cat subs.subfinder.txt subs.amass.txt subs.crtsh.txt | sort -u > subs.all.txt
```
Filter against scope before continuing:
```bash
grep -viEf $P/out-of-scope.regex subs.all.txt > subs.scoped.txt
```
(`out-of-scope.regex` comes from `scope_guard.py regex --write`: one extended regex per out-of-scope pattern in the scope block. Each matches the excluded host alone on a line and as the host of a URL — optional scheme or `//`, userinfo, a trailing dot and a port — followed by `/`, `?`, `#` or the end of the line; a path-level exclusion matches only URLs under its path. The patterns are lowercase, so the grep always takes `-i`.) The grep only removes excluded names; a name no in-scope pattern covers, and an IDN or fullwidth spelling of an excluded one, still pass. After §2, `python3 -I $S/scripts/scope_guard.py filter --engagement $P --write` holds `live-hosts.txt` to the in-scope patterns and records what it dropped.

### 2. Resolve and probe
```bash
dnsx -l subs.scoped.txt -silent -a -cname -resp -o resolved.txt
httpx -l subs.scoped.txt -sc -title -td -ip -cdn -fhr -rl $RL -H "$HDR" -o live.txt   # -fhr: same-host redirects; -fr carries $HDR to any host a redirect names
httpx -l subs.scoped.txt -p 80,443,81,591,2082,3000,3001,5000,7001,8000,8008,8080,8081,8443,8888,9000,9090 \
      -sc -title -td -rl $RL -H "$HDR" -o live-ports.txt   # the default probe is 80/443 only
cut -d' ' -f1 live.txt | sort -u > live-urls.txt
sed -E 's#^https?://##; s#[:/].*##' live-urls.txt | sort -u > live-hosts.txt
```
- **A service on a non-default port is a different application.** The default `httpx` probe
  is 80/443, so an admin console, a metrics endpoint, a dev build or a management UI on
  `:3000`/`:8080`/`:8443` is invisible to it — and those are the ones with no edge in front
  and often no authentication. Merge `live-ports.txt` into `live.txt`; each host:port pair
  is its own row in `map.md` and its own `surfaces.md` implementation, because it is usually
  a separate codebase. Scope rules apply to the port too: a host in scope does not put a
  non-standard port in scope if the scope table names a port (`SKILL.md` → hard gate 1).
- Hosts with a CNAME but no A/AAAA record, or a CNAME pointing at a cloud service returning a "not found" page → takeover candidates (`vectors-server.md`).
- Group `live.txt` by tech (`-td`) into `map.md`.
- **Frameworks:** confirm the stack and version per host from the signals in `vectors-frameworks.md` → Fingerprints, and the front end in front of it; record both in `map.md` → Frameworks.
- **Functional siblings:** walk signup, login, and the main app once with Burp recording and note every first-party host the flow actually uses — redirects, form actions, XHR/fetch, WebSocket URLs (`auth.`, `id.`, `api.`, `account.`). Record each with the flow evidence. Ignore analytics, CAPTCHA, CDN, payment, and support providers. A sibling found this way is target surface only if the scope table covers it; otherwise ask the program (`SKILL.md` → hard gate 1).

### 3. URL and endpoint corpus
```bash
gau --subs $D > urls.gau.txt            # passive: an archive query, no request to the target
waybackurls $D > urls.wayback.txt       # passive
# ACTIVE. -cs is the control that matters; -fs alone does NOT hold it (see the warning below)
katana -list live-urls.txt -rl $RL -H "$HDR" -o urls.katana.txt \
       -cs '^https?://([a-z0-9_-]+\.)*target\.com(/|$)' \
       -cos '(halarastatic|halaracdn|doubleclick|facebook|onelink|awswaf)' \
       -fs fqdn -jc -d 3 -kf all
cat urls.*.txt | sort -u | grep -viEf $P/out-of-scope.regex > urls.all.txt
```

> **`-fs fqdn` does not keep katana in scope, and believing it does sends traffic to
> third parties.** Measured 2026-10-25: a run with `-fs fqdn -jc -kf all` against two
> in-scope storefronts produced 212 728 URLs of which **~94 000 were on hosts the program
> does not authorise** — 58 530 on the asset CDN, 24 657 on the image CDN, 4 308 on an
> AWS WAF captcha host, 3 458 on out-of-scope regional storefronts reached through a geo
> redirect, 1 641 on `www.facebook.com`. `-fs` sets which *field* the default host scope
> is computed from; `-jc` then follows script sources and `-kf` known files wherever they
> live, and a 30x is followed off-host. **`-cs` (in-scope regex) is the flag that
> constrains what is followed**, `-cos` excludes the rest, and redirects still need
> watching.

**Adjudicate the corpus before anything consumes it.** The crawl is discovery, so it will
contain hosts the program has not authorised; that is expected. What is not acceptable is
*requesting* them, or letting them reach `shapes.txt` and the tracker:
```bash
python3 -I $S/scripts/scope_guard.py check --engagement $P $(
  awk -F/ '{print $3}' urls.all.txt | sort -u)         # every host, adjudicated
# keep only URLs whose host passes, then shape THAT file
python3 -I $S/scripts/shapes.py urls.in-scope.txt > shapes.txt
```
Record the dropped set in `recon/urls.dropped-not-in-scope.txt` and say in the handoff how
many hosts it covered. A host discovery found and the program has not authorised is an
operator-queue item, never an assumption.
Classify the corpus:
```bash
grep -Ei '\.(js|map)(\?|$)'          urls.all.txt > urls.js.txt
grep -Ei '[?&](url|uri|redirect|next|return|dest|callback|continue|target|link|image|src|feed|webhook)=' urls.all.txt > urls.ssrf-redirect.txt
grep -Ei '[?&](id|uid|user|account|org|order|invoice|doc|file)=' urls.all.txt > urls.idor.txt
grep -Ei '[?&](q|search|query|filter|sort|order|lang|template|page)=' urls.all.txt > urls.inject.txt
grep -Ei '\.(bak|old|orig|swp|zip|tar\.gz|sql|env|log|conf|ya?ml)(\?|$)' urls.all.txt > urls.sensitive.txt
```
Collapse URLs into **path shapes** — the tracker tests one shape once, not every ID:
```bash
python3 -I $S/scripts/shapes.py urls.all.txt > shapes.txt            # {id} {uuid} {hash} {token}; ports kept; query dropped
#   with js-codex: add its bridge as a second argument (never merged into urls.all.txt)
#   python3 -I $S/scripts/shapes.py --collapse 20 urls.all.txt $P/recon/js-codex/<asset>/raw/js-codex/out/urls-current.txt > shapes.txt
python3 -I $S/scripts/shapes.py --count urls.all.txt | sort -rn | head  # the shapes with the most URLs behind them
```
`{token}` is a 24+ character segment of letters, digits, `_` and `-` with at least one letter and one digit that does not read as words (at most one run of 5+ lowercase letters, unless it has 5+ digits): signed IDs and share tokens collapse, `getUserNotificationSettingsV2` stays literal. Spec templates (`{userId}`, `{order_id}`, Postman `:orderId`) become `{id}`, so a spec and a crawl of one route give one shape; `/api/1/` and `v2` stay literal. `--methods` keeps a `METHOD url` prefix (any HTTP method, e.g. `PROPFIND`) for lists built from a spec or proxy history.

### 4. Content discovery
```bash
ffuf -u https://HOST/FUZZ -w <wordlist> -H "$HDR" -mc all -fc 404 -ac -rate $RL -o ffuf.HOST.json
```
Wordlists: SecLists `Discovery/Web-Content/raft-medium-*.txt` and `api/api-endpoints.txt` at this tier.

High-value paths to request directly on every live host:
```
/.git/HEAD  /.env  /.DS_Store  /server-status  /actuator  /actuator/env  /actuator/heapdump
/swagger.json  /openapi.json  /v2/api-docs  /v3/api-docs  /api-docs  /swagger-ui/  /graphql  /graphiql
/.well-known/openid-configuration  /.well-known/security.txt  /crossdomain.xml  /robots.txt  /sitemap.xml
/debug/pprof/  /phpinfo.php  /wp-json/  /console  /admin  /_next/static/  /static/js/*.map
```

### 5. Templated scanning (only if the program allows automation)
```bash
nuclei -l live-urls.txt -severity medium,high,critical -rl $RL -H "$HDR" -o nuclei.txt
```
Treat every hit as a lead. A nuclei result is not a report until you have reproduced it manually.

---

## L2 — Intermediate

### 6. JavaScript analysis

**First question of this section: where is the client's code served from?** Look at one in-scope
page's `<script src>` hosts before collecting anything. If the bundles come from a CDN or a
sibling subdomain — and on a large SaaS front end they almost always do — then the application's
own code is *unscoped* by hostname, and a collector that only trusts in-scope origins will store
the entire front end as third-party vendor code. The run then looks successful: artefacts are
stored, counts are healthy, gates pass. What is missing is the only thing that mattered — the
client's route table and request shapes, which is what correlates a front-end call to the surface
being tested.

With `js-codex`, name those hosts with `--code-host` (one per host) so they are analysed as the
target's own code, and record the same statement in the scope block's `static-code:` section so
the engagement's gate agrees: read their code, request nothing of them. Its acceptance gate
*"the target's own client code was analysed as the target's"* fails a collection where nothing
was treated as first-party, and prints the hosts to name. Without js-codex, the equivalent is to
point the jsluice/source-map pipeline at those hosts deliberately rather than at the in-scope
host list, which will not contain them.

Measured consequence of getting this wrong (2026-10-27): an authenticated pass on a suite product
stored 377 scripts, **all** of them classed vendor because every bundle came from
`*.larksuitecdn.com`, and recovered nothing of the authenticated client's request surface.
**With `js-codex` installed, it is this section.** It collects through a browser you lease for
the engagement (bodies from the DevTools Network domain, across iframes and workers, so no
second request), fetches only what the browser never loaded, and cites every claim to a byte
offset or to captured traffic. Its scope is `scope_guard.py`'s answer, its rate is `RL`, its
header is `HDR`, and it leaves from the `recon` exit:
```bash
J=<js-codex skill directory>
export JS_CODEX_H1_ENGAGEMENT=$P JS_CODEX_H1_SKILL=$S         # h1 mode: its h1scope adapter answers from scope_guard
export JS_CODEX_PROXY="$(python3 -I $S/scripts/egress.py proxy-for --engagement $P --role recon --host <in-scope host>)"
python3 -I -B $J/bin/browser-lease start --engagement $P --role anon --header "$HDR"   # the lease: CDP 9301, its own terminal
python3 -I -B $J/bin/js-codex run --target-root $P/recon/js-codex/<asset> --cdp 9301 --expect <in-scope host> \
    --header "$HDR" --delay "$(awk -v r="$RL" 'BEGIN{print 1/r}')" --flow $P/sensitive/flows/anon.flow
python3 -I $S/scripts/referenced_hosts.py harvest --engagement $P     # §6b: reads the edges it wrote
```
Its endpoints reach `shapes.txt` through the bridge file `to-ledger` writes (§3's `shapes.py`
line); a target root outside `$P/recon/` is refused.
Collection — `run --cdp`, `collect`, and with them `--probe-specs` and `--guess-maps` —
refuses without `automation: allowed` in `scope.md` → Rules (P0 step 8); without `--cdp` the
run is analyze-only, sends nothing and needs no ruling. Where each output
goes — URL corpus, `map.md` Secrets and leads, tracker flags, `docs-claims.md`, P5 — is the
table in its `SKILL.md` → *In an h1-app-test engagement*. **Its files are leads, not
evidence**: `coverage_shapes.py`, `asset_census.py` and the tracker's write rule ignore
everything it derived from bytes, and count only the traffic the browser really sent.

**Without it**, the pipeline below:
```bash
DELAY=$(awk -v r="$RL" 'BEGIN{print 1/r}')    # seconds between requests, from RL
mkdir -p js && while read -r u; do
  curl -s -H "$HDR" "$u" -o "js/$(echo "$u" | md5sum | cut -c1-12).js"; sleep "$DELAY"
done < urls.js.txt
```
- **Source maps:** for each bundle, try `<bundle>.js.map` (same loop, same `$DELAY`). If it exists, reconstruct the original sources and grep those instead — original source carries comments, variable names and dead code that the minified bundle does not.
  - `sourcemapper -url <bundle>.js.map -output src/` for a remote map.
  - `unwebpack-sourcemap -l --make-directory <file>.js.map out/` for one already on disk, or `-d <url>` to detect the map from a bundle. It is a single script with its own venv (`~/.local/share/unwebpack-sourcemap`, shim in `~/.local/bin`); the repo's `requirements.txt` pins 2019 versions, so install current `requests` and `beautifulsoup4` instead.
  - Never pass `--dangerously-write-paths`: the map controls the paths, so a hostile one writes outside the output directory.
- **Endpoints — `jsluice` is the primary for a directory of saved bundles:**
  ```bash
  jsluice urls js/*.js > js-urls.jsonl          # one JSON record per reference
  python3 -c 'import json,sys;print("\n".join(sorted({json.loads(l).get("url","") for l in open(sys.argv[1])}-{""})))' \
      js-urls.jsonl > js-endpoints.txt
  ```
  It parses the AST rather than matching strings, so it recovers URLs built by
  concatenation and reports the HTTP method where the call site shows one.
  **Measured 2026-10-25** on 80 production Next.js chunks (17 MB): 3.7 s, 3168 records,
  **1561 distinct URLs**, 155 API-shaped.
  - **Do not hand-roll the grep.** The same corpus searched with
    `grep -ohE '"[^"]*api/v1/[^"]*"'` returned 183 strings and **missed
    `/api/awin/order` entirely** — a real unauthenticated endpoint that a later run
    reported — because the pattern assumed every API path carries `/api/v1/`. The
    bytes were already on disk; the extractor was the failure. A parser has no house
    style to assume.
  - **`xnLinkFinder` is for crawl mode (`-i <URL>` with `-d`), not for a bundle
    directory.** On this install (8.2) a directory of JS files produced **zero output,
    exit 0, empty files**, with `-sf` as a literal domain, as a file, with and without
    `-sp`/`-spo`, and with `-v`. Two further traps if you do use it: `-sf` became
    **mandatory** in 8.2, and the tool **blocks forever reading stdin** when stdin is a
    socket — a background run sat 58 minutes at 0% CPU before it was killed. Redirect
    `< /dev/null` always.
  - `trufflehog filesystem js/ --no-verification` still earns its place on the same
    directory; on that corpus `jsluice secrets` returned 0 and is worth running anyway,
    since the two detectors disagree often.
- **Secrets:** `trufflehog filesystem js/ --no-verification` and `jsluice secrets js/*.js` — two detectors with different coverage; a hit in one and not the other is normal. Add `xnLinkFinder -os` only if you are already running it in crawl mode. `--no-verification` keeps trufflehog from sending each secret to its provider; hits go to `map.md` → Secrets as `unverified`, and the liveness check waits for P4 (`reporting.md` Gate 5).
- **Things to grep for by hand:** `admin`, `internal`, `debug`, `role`, `isAdmin`, `featureFlag`, `apiKey`, `Authorization`, `postMessage`, `addEventListener("message"`, `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, `bypassSecurityTrust`, hidden routes in the router config.
- Admin/staff routes and API calls found in a regular-user bundle are BFLA candidates.
- **Tuned wordlists:** re-run section 4 with words harvested from the bundles plus the tech-specific list once the stack is known (Spring → `actuator`, Rails, Laravel, etc.).

### 6b. Referenced hosts — a strict scope does not make the asset self-contained

One host in scope is not one host of attack surface. That host loads script from somewhere,
sets a cookie for a parent domain, names origins in a CSP, trusts an origin in CORS, hands
an OAuth code to a redirect URI, posts messages into a frame, and calls an API. Somewhere
else there is usually a second copy of the same application — staging, UAT, a regional
build, an old marketing clone — serving the same data through the same code with fewer
controls in front of it. **Every one of those is an edge: a mechanism by which something the
program never listed decides what happens on the asset it did list.**

So an unlisted host is two separate questions, and only one of them is settled by the scope
table:

- *May it be requested?* Hard gate 1. Unknown → operator queue. Nothing changes here.
- *What can it do to the in-scope asset?* **This needs no permission to answer**, it is
  answered from the corpus already on disk, and it is the half that decides whether a finding
  is a report **against the listed asset** or a policy violation against an unlisted one.

The failure this section exists to prevent is mechanical, not conceptual: §1–3 drop
out-of-scope and unknown hosts into `recon/scope-guard-dropped.txt`, the pipeline moves on,
and nobody reads that file again. On a wildcard program that costs little. On a single-asset
program it discards most of what the asset actually depends on.

```bash
python3 -I $S/scripts/referenced_hosts.py harvest --engagement $P   # ledger, no requests
# fill the Edge and Ruling columns, then:
python3 -I $S/scripts/referenced_hosts.py check --engagement $P
```
It reads `recon/js/`, the saved headers and responses, `recon/*.txt`, the specs and
`map.md`; lists every host `scope_guard` does not place in scope; records which edge signals
appeared on the same line, with a `file:line`; and writes `recon/referenced-hosts.md`. Hosts
with no edge signal go to `recon/referenced-hosts-noedge.txt` and are a count, not a row.
Re-running it after a later crawl rewrites the machine columns and **keeps** what you wrote.
`tracker_check.py --phase P3/P4` fails while any row has no edge or no ruling, and fails when
hosts were dropped and no ledger was ever built.

#### The edges, and what each one buys on the in-scope asset

| Signal | What it means | What a bug there does to the asset in scope |
|---|---|---|
| `serves-code` | `<script src>`, `importScripts`, `Worker`, stylesheet, `@import`, iframe `src`, importmap | Control of that host — takeover of a dangling name, a writable bucket, an upload — is **script execution in the asset's origin**. No SRI makes it direct |
| `csp` | the host is named in `script-src` / `default-src` / `connect-src` | An open redirect, a JSONP endpoint or a file upload there is a way **through the asset's own CSP**, which is also what turns the asset's "unexploitable" HTML injection into XSS |
| `cors` | the asset answers `Access-Control-Allow-Origin: <host>`, with credentials | An XSS there **reads the asset's authenticated responses**. The asset's CORS policy is a list of hosts whose security is now the asset's security |
| `cookie-domain` | `Domain=.target.com` on the asset's session cookie | Every sibling of the asset can **read and set the asset's session**. One XSS anywhere under that parent is account takeover on the listed host |
| `redirect-allowlist` | `redirect_uri`, `return_url`, `returnTo`, `continue`, `RelayState` accepting that host, or a `*.target.com` whitelist | An open redirect there is **token or code theft** for the asset. This is the single most-paid edge on a strict scope, because the login flow is on the listed host and the redirect is not |
| `postmessage` | `postMessage` target, or an `origin` comparison naming it | A frame on that host **drives the asset's client state**; a sloppy origin check (`startsWith`, `indexOf`) extends that to anyone |
| `api-call` | the asset's `fetch`/XHR/`baseURL` points there | It **is** the asset's API. An authorization bug there is the asset's authorization bug, whatever the hostname says |
| `auth` | `oauth`, `/authorize`, `openid`, `saml`, `/sso`, `client_id` | Account takeover there **lands on the asset** |
| `sibling-name` | shares its last two labels with an in-scope host | First-party, and the program did not list it. Ask |
| `env-name` | `staging`, `uat`, `dev`, `qa`, `sandbox`, `preprod`, `internal`, `legacy`, `old` in the name | Usually the **same application with the controls turned down**, and frequently the same database. Authentication relaxed for testing, debug endpoints left on, an index page listing builds, a `.git`, a `/actuator` — plus unreleased features the production host has not shipped |

The last two are the ones a single-asset scope most often hides, and they compound: a
`sibling-name` + `env-name` host that also carries `api-call` is the production API with its
guards off.

#### Turning an edge into something that pays

1. **Write the edge as a sentence about the in-scope asset**, mechanism first — "the session
   cookie is scoped to the parent, so XSS on any sibling takes over accounts on the listed
   host", not "staging is exposed". If the sentence cannot be written, the ruling is
   `no-edge` and that is a real answer: record it and stop thinking about the host.
2. **Collect the proof of the edge from the in-scope asset itself**, which needs no ruling:
   the `Set-Cookie` with the parent domain, the CSP header, the `<script src>`, the
   `redirect_uri` the asset accepts, the API host its own bundle names. That evidence is on
   the listed host and it is where the report's severity comes from.
3. **Then the ruling, and only then the request.** The operator decides: in scope (quote the
   table), excluded (quote the rule), ask the program (queue number in the row), or leave it.
   A host whose edge is `serves-code` — the asset's own bundles on an unlisted CDN or
   `static.` host — has one more answer, `static-code`: its code files may be **read**
   (GETs of `.js`, `.mjs`, `.cjs`, `.map`, through js-codex only) and nothing else is ever
   sent to it. The ruling goes in the row *and* as an entry in the scope block's
   `static-code:` section (`scripts/scope_guard.py` docstring), then `scope_guard.py init`;
   `referenced_hosts.py check` fails while the two disagree, and `init` fails an entry the
   program excludes. It is never scope: `scope_guard.py check` still answers no for that host.
   A strict policy and a demonstrable edge is exactly the case worth asking about, because
   the question is not "may I test an extra host" but "this host decides what happens on
   yours — is it yours?" Programs that list one domain do rule these in; a few do not, and
   the ruling is the difference between a paid report and a closed one.
4. **The report's asset is the listed one.** The unlisted host is the mechanism in the
   reproduction steps, never the headline. A title that names a host the program does not
   recognise gets routed as out of scope before anyone reads the impact.
5. **Sweep the edge, not the host.** One `*.target.com` redirect allowlist means every
   sibling is a candidate; one parent-domain cookie means every sibling is; one host in the
   CSP means every host in the CSP (`SKILL.md` → Technique tiers, *a filter is a signal to
   sweep*).

#### Two false-positive classes, both cheap to rule out

- **Your own payloads.** The harvest reads `requests/`, so the `Origin: https://evil.example.com`
  you sent in a CORS probe comes back as a row with a `cors` signal. It is an origin the asset
  *answered*, not one it depends on — ruling `no-edge`, reason "our own probe", and the row
  stays as the record that the probe happened.
- **Vendor hosts with a real edge.** An analytics or tag-manager host in `script-src` genuinely
  can execute script in the asset's origin; that is why it is a row. The ruling is still
  usually `excluded` with the third-party rule quoted. Quote it rather than deleting the row,
  so the next engagement on this program does not re-derive it.

Measured on a live engagement: 45 files, 83 hosts not in scope, **13 with an edge signal** —
among them the apex the session cookie is scoped to, and two `qa.`/`test.` hosts in a sibling
domain family where the scope lists six names and no wildcard. The other 70 were a count.

#### Where else these hosts come from

The harvest reads what is on disk, so it only sees what earlier sections saved. Feed it:
§2's functional-sibling walk (every first-party host the signup and login flows actually
use), §3's archive corpus, §4's `/crossdomain.xml` and `/.well-known/`, the response headers
from §2 (save them — CSP, CORS and `Set-Cookie` are the three highest-value sources and a
`-sc -title` probe does not keep them), §8's specs, §9's app strings, and §13's historical
bundles, where a host that has since been removed from production JS still resolves.

### 7. Parameters
Six independent sources. Four are already covered elsewhere in this file; take all six,
because a name that one source misses another usually has.

1. **HTML attributes** — `id`/`name` on every element, not only form inputs (the P2 walk).
2. **JavaScript** — `jsluice` output and the §6 harvest; variable names are parameter
   candidates even when the call site builds the query dynamically.
3. **Archives and indexes** — `urls.all.txt` from §3 already holds every parameter the
   Wayback corpus and the crawl saw; `grep -oE '[?&][a-zA-Z0-9_-]+=' urls.all.txt | sort -u`.
4. **Fuzzing** — `x8` (calibrated, see §0c item 13), `arjun`, Burp **Param Miner**
   ("Guess params" / "Guess headers"), `ffuf` filtered on size *after* measuring the
   page's own variance.
5. **Re-use across endpoints** — take every parameter seen anywhere on the target and try
   it on the *other* endpoints. Teams reuse names across services; a parameter the cart
   honours is worth sending to the order service. This is the cheapest of the six and the
   one most often skipped.
6. **Mobile and partner clients** — parameters in app source or partner docs that the web
   client never sends.

**Fuzz GET and POST separately — they are not the same search.** Back ends bind query and
body parameters through different code: different parsers, different validators, often
different framework hooks, and a name ignored on the query string is frequently honoured in
the body (and the reverse). `x8` takes `-X POST --body '{}'`; Param Miner has separate
"Guess params" targets for query, body and cookies. Doing one and reporting the other as
covered is the commonest way a parameter sweep reads complete while half of it never ran.
The same split applies to **cookies** and **headers** (`x8 --headers`), which is a third
and fourth binding path.

A custom wordlist built from 1–3 and 5 beats a generic list; keep both and concatenate.

- Note every parameter that the UI never sends but the API accepts → mass assignment candidates.
- `x8 --headers` switches the same calibrated search to header discovery.

### 8. API surface

**Finding the spec is its own job, and `references/docs-mining.md` §A is how it is done** —
the host's own default paths per framework, the docs site made to hand over the file it
renders, the published SDK read as a generated client, Postman public workspaces, code
search, the changelog, and the archive. §4's eight paths are the start of that list, not the
whole of it. Then, separately, §B of the same file: the vendor's **human** documentation
states security promises in writing, and testing those sentences is the cheapest
high-confidence work on a mature program (§0c item 14, `docs-claims.md`).

- Save every OpenAPI/Swagger or Postman spec under `$P/recon/api-specs/`, then turn it into scoped operations:
  ```bash
  python3 -I $S/scripts/api_contract.py --engagement $P --spec $P/recon/api-specs/<file> --shapes [--base-url https://<in-scope base>]
  ```
  It rewrites `recon/api-contract-candidates.jsonl` (method, URL, operation, auth hint, object-reference parameters and body fields, sink-like names, write or read, priority): this spec's earlier rows are replaced, other specs' rows kept, duplicates dropped. `--shapes` merges the operations' shapes into `recon/shapes.api.txt` (sorted, unique, already in `shapes.py` form). Every URL is checked as the final absolute URL (base + path, every server and server-variable value, Swagger 2 `schemes`/`host`/`basePath`) against `scope_guard` hosts and path rules; out-of-scope, unknown and host-moving URLs are listed on stderr and never written. Relative servers need `--base-url https://<in-scope host>`; zero in-scope operations exits 2 (empty population): note it in the P1 handoff. Write operations with object references are the first tracker rows for IDOR write. Also import the spec into Burp or Postman and hit every operation as each role (P2).
- GraphQL: try introspection. If it is disabled, use `clairvoyance` (field-suggestion abuse) to rebuild the schema. Load the schema into Burp InQL.
- Look for older API versions (`/v1/` when the app uses `/v3/`), mobile API hosts (`m-api.`, `mobile.`), and internal-looking hosts (`staging`, `dev`, `uat`, `internal`) — only if in scope.
- **WebSockets:** list every `ws://`/`wss://` endpoint from Burp history and bundles (`grep -Eo 'wss?://[^"'\'' ]+' js/*.js`), plus Socket.IO (`/socket.io/?EIO=4&transport=polling`) and GraphQL subscription endpoints. Each one goes to the tracker for CSWSH and per-message authorization.

### 8b. Authenticated re-crawl, and the flows the crawl cannot reach
Repeat sections 3, 6, and 8 while logged in as each role. With `js-codex`, that is one lease per role — a browser profile logged in as A, another as B, never both in one — each role's run into its application's target root, so later passes merge by content hash. **Per listed web application**: every app in `surfaces.md` → Assets that serves its own client gets its own pair, `recon/js-codex/<asset>/` for the anonymous pass and `recon/js-codex-auth/<asset>/` for the authenticated one, because a crawl driven from one app structurally never loads another app's chunks. `<asset>` is the row's text, its host, or the host with dots as dashes (`app-a` for `app.a.test`); `recon_coverage.py` resolves it against the Assets rows and fails a name that fits none or fits more than one. A host with no client of its own — an API, a stream, a protocol surface — is `out of play` in that table with the reason, and its paths come from the corpus of whichever application calls it. Authenticated bundles, settings pages, and API responses reveal hosts, internal URLs, feature flags, and admin routes the anonymous crawl never saw. Diff the results against the anonymous run; everything new goes to `map.md`.

**A crawl only harvests the pages it can reach, and the most interesting chunks are behind a completed flow.** Code-split bundles load per route, so an order-confirmation page, a post-purchase review form, a returns or support thread, and a settings sub-page each pull chunks no amount of crawling the storefront will reference — and those are exactly the chunks that carry payment, affiliate, fulfilment and staff-facing calls. Walk each money and identity flow **to its terminal page** (checkout → confirmation, order → detail → return, review → submit, ticket → thread) with the capture running, and harvest §6 from *those* pages too. Where a flow cannot be completed — no payment instrument, no live promotion — say so per flow in `map.md`, because "not in the bundle corpus" and "not reachable from the pages I loaded" are different claims and only the second one is usually true.

### 9. Mobile apps — static (only when listed as assets)
```bash
apktool d app.apk -o app_src && jadx -d app_jadx app.apk
grep -rEoh 'https?://[a-zA-Z0-9./_-]+' app_jadx | sort -u > $P/recon/mobile-endpoints.static.txt
```
Check `AndroidManifest.xml` for exported components and deep links, and `res/values/strings.xml` for keys.

This section only harvests what the web/API test needs: endpoints, hosts, keys, API versions. The app itself — exported components, deep links and App Links, WebView bridges, local storage, iOS URL schemes — is tested with the `mobile-app-security` skill when the app is a listed asset; its mobile-API leads come back here as tracker rows.

### 9b. Correlate the app's API with the web's — the delta is the target

A flat list of the app's endpoints is not the point of pulling the app apart. **The point is
the diff.** The web front end and the app call the same business logic through different
handlers, usually written by different teams at different times, and a check added to one
is frequently absent from the other. That is why the app's API is the standard place to find
an authorization bug the web path refuses — not because mobile code is worse, but because
*two implementations of one rule* is the condition the bug needs.

So the unit of work here is **the operation seen from both clients**, and the three sources
feed one comparison:

```bash
# A — the web corpus you already have (§3), normalised to shapes
#     recon/shapes.txt

# B — what the app ships: static strings, then what it actually sends. Absolute paths:
#     this file cd'd into $P/recon, and channel_diff resolves relative paths against $P.
#     JADX is the sub-engagement's decompile when mobile-app-security ran P1, else §9's.
JADX=$P/mobile/<package>/static/jadx/sources                         # or app_jadx from §9
grep -rEoh 'https?://[a-zA-Z0-9./_:-]+' "$JADX" | sort -u            > $P/recon/mobile-endpoints.static.txt
#     then proxy the app (§15) and export Burp's sitemap/HAR for that host into
#     $P/recon/mobile-endpoints.proxied.txt: every URL the app really calls, including
#     the one-time flows (surfaces.md). mobile-app-security P2 step 7 writes the same file
cat $P/recon/mobile-endpoints.static.txt $P/recon/mobile-endpoints.proxied.txt 2>/dev/null \
    | sort -u > $P/recon/mobile-endpoints.txt

python3 -I $S/scripts/channel_diff.py --engagement $P \
        --a recon/shapes.txt --b recon/mobile-endpoints.txt --paths-only --list \
        | tee recon/channel-diff-$(date +%F).txt
```

`--paths-only` is usually right: the app often talks to its own API host, and the same
operation on two hosts is still one operation. Run it both ways — with the host and without
— because a *host* difference is itself a finding when one of the two has no edge in front.

**Read the three buckets differently. They are not three lists of the same thing.**

| bucket | what it is | what to do |
|---|---|---|
| **BOTH** | the operation exists on both sides. Existence matches; **enforcement is the open question** | Replay the operation through each client's shape, A against B's object, and compare what each one *requires*: auth scheme (cookie vs bearer vs API key), the headers it insists on, the signature, the parameters it accepts, the API version, the error format. This is the bucket that pays, and the comparison is the deliverable |
| **ONLY-WEB** | the app never calls it | Lower priority, with one exception: a *legacy* path the app still ships a code path for but no longer uses. Those are live and unmaintained |
| **ONLY-MOBILE** | no web client calls it | Usually the oldest and least reviewed handlers on the target: deprecated flows kept alive for shipped app versions, device and push registration, first-launch and consent calls, debug or feature-flag endpoints. No web-side test has ever touched them, because no web-side crawl can see them |

**What a BOTH row does not tell you.** Identical shapes are not identical requests — that is
exactly the point. The script says *where* to compare; it never says the comparison was
made. Record each one in `surfaces.md` → *Operation-level comparison*, and when the two
clients genuinely agree, say so there: a control that holds on both paths is worth knowing
and worth reporting as held.

**Both directions, not one.** The web test learns the API's hidden surface from the app, and
the app test learns from the web what the API *should* enforce. A parameter the web client
sends that the app omits is a parameter the server may be defaulting; a header the app sends
that the web does not (`X-App-Version`, `device-id`, a signature) is a value the server may
trust. Feed each into `§7` parameters and `vectors-logic-api.md` → Mass assignment.

**Cross-surface rendering.** Where an input exists on both sides, seed it through one client
and read it back through the other (`h1-invariant-hunt` step 8). One sink is usually hardened
— the one the developer was looking at — and the other is not. Mark each marker so you can
tell which client's input reached which renderer.

Scope applies to the app's list exactly as to any other: it is full of third-party SDK hosts
(analytics, crash reporting, ad networks, push), and `channel_diff.py` drops anything that is
not `in-scope` and says how many it dropped. Never send the research header to those hosts
(hard gate 2).

---

## L3 — Advanced

Anything found here is a **lead, not scope**. Re-check every new host and IP against `scope.md` before sending it a single HTTP request. IP-only assets are usually in scope only when listed explicitly.

### 10. DNS brute force and permutations
Queries go to public resolvers, not to the target's web servers. `puredns` needs `massdns` installed and filters wildcard DNS itself.
```bash
# resolvers.txt: a fresh, validated public resolver list (e.g. trickest/resolvers)
# DNSRL: total queries per second; low hundreds at most unless the program says otherwise
puredns bruteforce best-dns-wordlist.txt $D -r resolvers.txt -l $DNSRL -w subs.brute.txt
cat subs.scoped.txt subs.brute.txt | alterx -silent | puredns resolve -r resolvers.txt -l $DNSRL -w subs.perm.txt
cat subs.brute.txt subs.perm.txt | grep -viEf $P/out-of-scope.regex | sort -u - subs.scoped.txt -o subs.scoped.txt
```
Re-run section 2 on the new names. Permutations of existing names (`api-v2-staging`, `admin-eu`) surface hosts that never appeared in certificate logs.

### 11. Infrastructure pivots
```bash
tlsx -l live-hosts.txt -san -cn -silent -o tls-names.txt                        # extra hostnames from certificate SANs
httpx -l live-urls.txt -favicon -silent -rl $RL -H "$HDR" -o favicons.txt        # mmh3 hash → Shodan: http.favicon.hash:<hash>
asnmap -d $D -silent -o asn-ranges.txt                                          # verify: recent asnmap builds may require a ProjectDiscovery API key
```
- Favicon hashes and unique page titles find the same app on hosts that do not carry the target's hostname (origin servers behind a CDN, forgotten staging boxes).
- An origin IP that serves the production app directly bypasses the CDN/WAF — test it only if the IP or its hostname is in scope.

### 12. Virtual hosts
Name-based vhosts that exist on an in-scope server but not in public DNS (`admin.`, `staging.`, `internal.`):
```bash
ffuf -u https://<in-scope-ip-or-host>/ -H "Host: FUZZ.$D" -H "$HDR" -w vhost-words.txt -ac -rate $RL -o vhosts.json
```
Build `vhost-words.txt` from `subs.all.txt` prefixes, JS-derived hostnames, and permutations.

### 13. Historical JavaScript
Old bundles reference routes and API calls that were removed from the UI but often still work server-side.
```bash
mkdir -p js-hist
curl -s "https://web.archive.org/cdx/search/cdx?url=$D&matchType=domain&filter=mimetype:.*javascript&collapse=digest&fl=timestamp,original" > js.history.txt
while read ts u; do
  curl -s "https://web.archive.org/web/${ts}id_/$u" -o "js-hist/$ts-$(echo "$u" | md5sum | cut -c1-12).js"; sleep 1
done < js.history.txt
```
Run section 6's endpoint extraction over `js-hist/` and diff against `js-endpoints.txt`. Endpoints only in the old set go to the top of the test queue.

### 14. Code and leak search
```bash
trufflehog github --org=<github-org> --no-verification --json > leaks.github.json
trufflehog docker --image=<org>/<image> --no-verification --json > leaks.docker.json
```
Hits are `unverified` leads, as in §6.
- GitHub code search by hand: `"api.target.com"`, internal hostnames and header names from `map.md`, distinctive error strings, `"target.com" (token OR secret OR password)`.
- Public Postman workspaces and collections for the target's API.
- Report a secret only if it is live and grants access to an in-scope asset; check program policy on employee and third-party repositories. Verification rules: `vectors-server.md` → secrets.

### 15. Mobile apps — dynamic (only when listed as assets)
```bash
frida-ps -Uai                              # find the package name
objection -g <package> explore             # then: android sslpinning disable   (objection flags vary by version — check --help)
```
Proxy the app through Burp. Mobile APIs often lag the web app on authorization, MFA, and rate limiting — feed every new endpoint into `vectors-auth.md`.

When `objection` is not enough — Frida version matching, pinning in Flutter or React Native, apps that ignore the system proxy, request-signing headers — use `mobile-app-security` → `references/lab.md` and `references/traffic.md`.

### 16. Change monitoring
Snapshot what the JS bundles, API specs, and GraphQL schema contain every session, and diff that. Bundle file names change with every build, so diff the extracted lists, not the files:
```bash
SNAP=snapshots/$(date +%F); mkdir -p $SNAP     # not S: $S is the skill directory
SPEC=<spec saved in §8>.json; GQL=<introspection saved in §8>.json
jsluice urls js/*.js | jq -r '.url' | sort -u > $SNAP/js-endpoints.txt
jq -r '.paths | keys[]' "$SPEC" 2>/dev/null | sort -u > $SNAP/spec-paths.txt
jq -r '.data.__schema.types[] | select(.fields) | .name as $t | .fields[] | "\($t).\(.name)"' "$GQL" 2>/dev/null | sort -u > $SNAP/gql-fields.txt
PREV=$(ls -d snapshots/*/ | sort | tail -2 | head -1)          # the previous snapshot; the same as $SNAP on the first run
for f in js-endpoints spec-paths gql-fields; do comm -13 "$PREV$f.txt" "$SNAP/$f.txt" | sed "s/^/new $f: /"; done
```
New endpoints, fields, roles, and flags go straight to the top of the queue (P4 → *Session start*). `h1-invariant-hunt` step 10 does the same with its own layout when it is installed.

---

## Output to `map.md`

**Information gathering** — the fourteen items of §0c, one line each: the answer, or `n-a`
with its reason. A blank in that list is the gap this section exists to prevent.
Live hosts by tech · frameworks, versions, and front ends per host, with evidence · functional siblings with flow evidence and scope status · path shapes · auth model per host · **auth surface** (registration URL and method, verification method, CAPTCHA/anti-bot vendor, MFA, SSO providers, roles and plans, how many accounts the test needs) · API specs found · WebSocket endpoints · interesting endpoints by vector class · leaked IDs · secrets (with verification status) · takeover candidates · L3 extras: origin IPs, vhosts, endpoints only seen in historical JS.

Recon notes leads; it does not test them. A JS hint, an API spec entry, or a mobile-app endpoint is a hypothesis for Phase 3 — it stays on the tracker until it has a verdict, even if the browser never triggered it.
