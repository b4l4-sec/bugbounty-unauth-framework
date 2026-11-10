# Authentication and Authorization

Accounts: **A** = attacker, **B** = victim, both yours. Keep a Burp project per role, or use the Burp Autorize extension with B's session in Autorize and A's session in the browser.

Every section runs **L1 → L2 → L3**. L1 is the direct check. L2 bypasses the defence L1 ran into or reaches the same thing another way. L3 goes after disagreements between components, races, and protocol edge cases.

**Order matters here.** Finish every cross-account check while both accounts are healthy. Session, reset, password/email change, MFA counters, passkey add/remove, and races run last, on **B only** (`phases/P3-sweep.md` → disruptive tail). Stop at the first availability signal (`429`, `Retry-After`, CAPTCHA, "N attempts left", unexpected `401`, lockout email); record `partial` and move on. Identity-sensitive counters (login, OTP, reset, resend) get the smallest burst that answers the question: by default at most three invalid attempts on B, or one below an advertised lockout threshold; up to ~30 only when the program explicitly allows rate-limit testing. Never spray real users.

---

## Registration
**L1 — Basic**
- [ ] Use the app without verifying the email. Which features unlock?
- [ ] Mass assignment on signup: add `role`, `is_admin`, `org_id`, `verified`, `plan` to the body.
- [ ] Email verification token returned in a response, or still valid after use.

**L2 — Intermediate**
- [ ] Register an email that differs from an existing one only in case, a trailing dot, or `+tag` → collides with or takes over B.
- [ ] Verification token not bound to the account or address; predictable tokens.
- [ ] Domain-based auto-join: an unverified address on B's company domain joins B's org.
- [ ] **Pre-account takeover:** register B's email with a password before B signs up via SSO. Does B's SSO login merge into your account?

**L3 — Advanced**
- [ ] Unicode normalisation collisions: homoglyph/compatibility forms (`ﬀ`, `İ`, full-width characters). Find where signup, login, SSO linking, and the mailer normalise differently (`h1-invariant-hunt` → parser disagreement).
- [ ] The other pre-hijacking variants (Sudhodanan & Paverd, 2022): the attacker's session survives after B claims the email; trojan identifier (attacker adds a second email/phone they control before B takes over the account); unexpired email change; an IdP that does not verify emails.
- [ ] Uniqueness race: register the same address twice in one single-packet group (`vectors-logic-api.md`) → two accounts sharing one identity.

## Login
**L1 — Basic**
- [ ] SQLi/NoSQLi in username and password (`vectors-injection.md`).
- [ ] Response manipulation: flip `"success":false` / 401 → 200 in Burp. If the app proceeds, check whether the server issued a valid session or only the client UI changed (only the first one is a bug).
- [ ] Credentials sent in GET parameters or logged in responses.

**L2 — Intermediate**
- [ ] **Pre-auth endpoints** (login, activate, verify, reset, invite-accept, "check email"): do any return real data — profile fields, tokens, org details — before a session exists, or accept an ID and act on it?
- [ ] Lockout/rate limit on login: only if the program allows rate-limit testing, with a small burst (≤ 30 requests, on B). A missing limit alone is a hardening miss (`reporting.md` → Gate 5).
- [ ] Lockout keyed on something you control: rotate `X-Forwarded-For`/`X-Real-IP`, vary username case or add trailing whitespace, alternate valid and invalid attempts.
- [ ] Alternate login paths with weaker controls: mobile API, legacy `/v1/` login, OAuth password grant, basic auth on API hosts.

**L3 — Advanced**
- [ ] GraphQL alias/array batching to make many attempts in one rate-limited request (`vectors-logic-api.md`).
- [ ] Single-packet race on the attempt counter: N parallel attempts counted as one.
- [ ] Timing side channel that distinguishes valid users — rarely paid alone; log it as a lever.

## Password reset
**L1 — Basic**
- [ ] Token returned in the response body.
- [ ] Token expiry, single use, invalidated after a password change or after a newer token is issued.
- [ ] Reset flow sets the new password via a user ID in the final request → change the ID to B's.
- [ ] Sessions not revoked after a reset — a revocation record.

**L2 — Intermediate**
- [ ] **Host header poisoning:** send the reset with `Host: your-oast-host` or `X-Forwarded-Host: your-oast-host`. Does the emailed link point at your host?
- [ ] **Parameter pollution:** `email=b@x&email=a@x`, `email[]=b@x&email[]=a@x`, JSON `{"email":["b@x","a@x"]}`, `email=b@x%0a%0dcc:a@x`, `email=b@x,a@x`. Does A receive B's token?
- [ ] Token not bound to the user: use A's token on B's reset form.
- [ ] Token leaks: in a `Referer` header to third-party scripts on the reset page, in analytics URLs.
- [ ] Token as another type, or `null` for an account with no reset pending (`vectors-logic-api.md` → Type confusion).

**L3 — Advanced**
- [ ] Dangling-markup poisoning: when only part of the link is injectable (e.g. a port in `Host: target.com:'<a href="//OAST/?`), an unclosed attribute in the HTML email can carry the token to you when the email is rendered or scanned.
- [ ] Normalisation mismatch: request a reset for a Unicode-compatible form of B's address. The account lookup normalises; the mailer sends to the raw input you control.
- [ ] Time-derived tokens: request resets for A and B in one single-packet group and compare the tokens; collect many of your own tokens and look for structure (timestamp, sequential, `md5(email+time)`).
- [ ] Reset on every path (`h1-invariant-hunt` → worked shape): web, mobile API, GraphQL mutation, support-initiated reset, the GET on the emailed link (does it log in automatically?).

## Magic links and email-code login
Passwordless login by an emailed link or code. The token is the whole credential, so it gets every *Password reset* test plus the ones below. A link that logs in whoever opens it, from any browser, is the design — not a finding.

**L1 — Basic**
- [ ] Single use: open the link, then open it again in a fresh browser profile.
- [ ] Expiry: use it after the lifetime the email states; use an older link after a newer one was requested.
- [ ] Token or code in the response to the "send me a link" request (body, headers, a redirect `Location`).
- [ ] Links still valid after a password change, logout everywhere, or an email change.

**L2 — Intermediate**
- [ ] Token not bound to the account: request a link as A, then change the `email`/`user_id`/`uid` parameter on the redeem request to B's.
- [ ] Host header poisoning and parameter pollution on the send request, as in *Password reset*: does the emailed link point at your host, or does A receive B's link?
- [ ] Emailed codes ("enter the 6-digit code we sent you"): the OTP rules in *MFA* apply — three attempts by default, ~30 only where rate-limit testing is allowed, maths against the keyspace.
- [ ] `next`/`redirect`/`returnTo` carried through the link: after login, does the app redirect off-site with the token still in the URL (it leaks in `Referer`), or through an open redirect (`vectors-client.md`)?
- [ ] Token as another type (`null`, `true`, `[]`) on the redeem request (`vectors-logic-api.md` → Type confusion).

**L3 — Advanced**
- [ ] **Requester-session binding:** in "we sent you a link — waiting…" flows, the tab that *requested* the link is logged in when the link is clicked anywhere. Request a link for B's address from A's browser; B clicks the genuine email in B's own mailbox; does A's tab now hold B's session? Gate 3: the victim clicks a real email they did not ask for — record whether the email names the requesting device or location, or shows a code to compare.
- [ ] Redemption race: the same link in one single-packet group → two sessions from a single-use token (`vectors-logic-api.md` → Race conditions). Usually low alone; a lever for the requester-binding case.
- [ ] Token structure and normalisation, as in *Password reset* L3: links for A and B in one packet; Unicode-compatible forms of B's address.

## MFA / 2FA
**L1 — Basic**
- [ ] Skip the MFA step: after the password step, request an authenticated page or API directly.
- [ ] Response manipulation on the OTP-verify response.
- [ ] OTP reuse.
- [ ] OTP as another type: `true`, `0`, `null`, `[]` (`vectors-logic-api.md` → Type confusion).

**L2 — Intermediate**
- [ ] OTP brute force: worth it only when the code space is small (4–6 digits) with a long or no expiry, and the program allows rate-limit testing — then measure with at most ~30 attempts on B (the cap above) and show the maths against the keyspace. Otherwise stay within the three-attempt default; a missing limit alone is a hardening miss.
- [ ] OTP from another account; OTP valid after a newer one is issued.
- [ ] Disable MFA, change the MFA device, or regenerate backup codes without re-authentication → CSRF/XSS chain.
- [ ] MFA enforced on web but not on the mobile API, the legacy API, or the OAuth token endpoint.

**L3 — Advanced**
- [ ] MFA step bound to the session, not the user: complete the password step as A, then point the MFA step at B (a user cookie or parameter on the verify request), or interleave two half-finished logins in one session.
- [ ] Single-packet race on OTP verification to exceed the attempt limit.
- [ ] Weaker fallbacks: recovery via email/SMS, support reset, "lost device" flow, SSO login that skips the app's own MFA.
- [ ] "Remember this device" token not bound to the user or forgeable.

## Session management
**The window matters as much as the fact.** A credential that keeps working for two minutes
after logout and one that keeps working for four hours are different findings and often
different severities. Measure it on the server's own clock, and measure the tail: queue the
later probes with `harness.py later add --in 15m|1h|4h --kind liveness <url>` and run them
next session with `later run`. One engagement's only finding sat unsubmittable across two
sessions for exactly this, because nothing could ask the question after the session ended.

**Revocation record.** A check that asks whether a credential stopped working — logout, refresh after logout, password or email change, reset completion, role or membership removal — needs a differential, not just the action. Save four requests as `requests/revoke-<check>-{baseline,action,replay,control}.txt`:
1. **Baseline:** the protected resource, reached with the credential before the action.
2. **Action:** the logout, change, reset, or removal, as driven.
3. **Replay:** the same credential against the same resource, after.
4. **Control:** the same request with no credential, or a made-up one — proof that this check can fail.

Verdict in the tracker cell: replay refused the way the control was → `dead` (revoked); replay still answered → `vuln` (survived) and a finding; no working control, or a replay answered from a cache or by a challenge page → `partial`. A screenshot showing the logout succeeded is not a record.

**L1 — Basic**
- [ ] Logout invalidates the session server-side — a revocation record. Sequence: request a protected resource and save it; snapshot the access and refresh tokens; log out normally; replay the access token; replay the refresh token; use any token the refresh returned. No refresh token → that step is `n-a`; one that exists but was not exercised → `partial`.
- [ ] Session tokens in URLs, and whether they leak in `Referer` to third-party scripts or analytics.

**L2 — Intermediate**
- [ ] Session fixation: does the session ID change at login?
- [ ] Password change and email change revoke other sessions — a revocation record, with a second session's token as the credential.
- [ ] Long-lived tokens that never rotate; refresh tokens that survive a password change.

**L3 — Advanced**
- [ ] **Cookie tossing:** from XSS or a takeover on any sibling subdomain, set a cookie with `Domain=.target.com` and a more specific `Path` to fixate a session or a CSRF cookie on the main app.
- [ ] Revocation lag: remove a role or membership, then replay the old session/JWT immediately — permission caches and signed claims stay valid for a while.

## Email / phone change
**L1 — Basic**
- [ ] Change the email without the current password or re-auth → combine with CSRF/XSS for ATO.

**L2 — Intermediate**
- [ ] Verification link sent to the old address vs. the new one; the change applies before verification.
- [ ] Change to an email that is already registered → account merge.

**L3 — Advanced**
- [ ] Multi-endpoint race: "change email to X" and "confirm with token" in one packet — the token is issued for one value and applied to another (`vectors-logic-api.md`).
- [ ] Pending change + SSO: set B's address as a pending (unverified) email on A, then sign in with an IdP asserting B's address.

## Email-address parser differentials
Registration L3 covers Unicode normalisation. The wider class: the validator or domain check reads one address, the mailer delivers to another. What it buys is a *verified* address on a domain you do not own — the key to domain auto-join, SSO routing by domain, and "verified @company.com" gates. Reference: Gareth Heyes, *Splitting the email atom* (PortSwigger Research, 2024).

Accounts on that domain fall under the exception to hard gate 3 in `SKILL.md`: only where the program's test-account rules allow them, each with a Cleanup ledger row. Proof: let a domain you control stand in for B's company (set B's org to auto-join it) whenever the app allows it. When the only domain that matters is the program's own, stop at the verified address and put it in the operator queue before using anything it unlocks. Every test address must deliver to a mailbox you control.

**L1 — Basic**
- [ ] Map every place the address is parsed: signup validator, domain allowlist/auto-join, SSO home-realm discovery, invite matching, the outbound mailer. Fingerprint the mailer from a mail the app sends you (`Received`, `Message-ID`, `X-Mailer`).
- [ ] Substring and suffix domain checks: `a@target.com.you.example`, `a@youtarget.com`.

**L2 — Intermediate**
- [ ] RFC 5322 forms the validator accepts but splits differently from the mailer: a quoted local part containing `@` (`"b@target.com"@you.example`), comments `(…)`, legacy source routes (`@target.com:a@you.example`).
- [ ] Where the mailer library is open source, reproduce its parsing locally first and note which recipient each form produces; take only the forms that split to the target.

**L3 — Advanced**
- [ ] RFC 2047 encoded-words in the local part (`=?utf-8?q?…?=`, `=?x?b?…?=`): the validator sees `…@target.com`; a mailer that decodes encoded-words can produce a different recipient (`=40` decodes to `@`, `=3e` to `>`). Charset choices (`utf-7`, unknown charsets) change what survives the filter.
- [ ] Unicode overflow: a code point above `0xFF` whose low byte is `@` (`0x40`) or `>`, truncated to one byte by the mailer.
- [ ] Malformed punycode (`xn--…`) labels that a library decodes into `@`, `<` or `>`.
- [ ] Chain the verified address to the promise it breaks: auto-join into B's org, SSO routed to the wrong IdP, access gated on a verified company domain (*Authorization — multi-tenant* → domain claiming).

## OAuth 2.0 / OIDC (as a client of an external IdP, or as a provider)
**L1 — Basic**
- [ ] `state`: missing, not validated, or not bound to the session → login CSRF / account-linking CSRF (attach A's social account to B).
- [ ] `redirect_uri` to an entirely different host.
- [ ] Scope escalation: request a broader `scope` than the UI does.
- [ ] **Press *Cancel* / *Deny* and see what is issued anyway.** The happy path gets the
  attention; the refusal path is a second handler and is often wired to the same
  redirect-and-issue code. Does the callback still receive a usable `code` or token? Is the
  cancel or return URL a parameter you control (`cancelUrl`, `denyUrl`, `error_uri`)? One
  click, and it is the cheapest OAuth test there is. The L3 *dirty dancing* item below is
  the general form of this; do the plain version first.

**L2 — Intermediate**
- [ ] `redirect_uri` validation: different path (`/callback/../redirect?url=`), subdomain, `@`, `#`, `?`, URL-encoded variants, a localhost/IP literal, a duplicated parameter, and chaining with an open redirect on an allowed host.
- [ ] `response_type` switch: `code` → `token` or `id_token token` to get the token in the fragment on a looser redirect.
- [ ] PKCE: remove `code_challenge` or downgrade to `plain`; use a `code` without a `code_verifier`.
- [ ] Authorization code: reusable, not bound to `client_id`/`redirect_uri`.
- [ ] `nonce` missing/not validated (OIDC implicit/hybrid).
- [ ] Account linking trusts an unverified email claim from the IdP (an IdP that lets you set any email) → log in as B.
- [ ] Token leaks through `Referer`, `postMessage` with `*` origin, or a third-party script on the callback page.
- [ ] **Device authorization grant** (RFC 8628 — TV, CLI, and IoT logins; `device_authorization_endpoint` in `/.well-known/openid-configuration`). The approval on the verification page needs CSRF protection and a consent screen that names the client. Start a flow as the attacker, then have B's logged-in browser open `verification_uri_complete` from your page: if it approves on `GET`, or the confirm form takes no CSRF token, your polling `device_code` receives B's tokens — one click. B is your own account: prove it with one identity call (`/userinfo` → B's `sub`), then revoke the grant. Device-code *phishing* (B types your code after a message you sent) is social engineering on most programs — check the policy.
- [ ] Device flow binding: `device_code` not bound to `client_id` (poll with another client's ID); `device_code` or `user_code` still usable after approval, denial, or expiry; a broader `scope` in the device request than the first-party client asks for.
- [ ] **Refresh tokens:** after a refresh, does the old refresh token still work (public clients — SPAs, mobile apps — should rotate or sender-constrain them, RFC 9700)? Is a refresh token issued to client X accepted with client Y's `client_id`, or without the confidential client's authentication? Does a refresh with a broader `scope` than the original grant succeed?

**L3 — Advanced**
- [ ] **Dirty dancing** (Rosén, 2022): break the flow on purpose — invalid `state`, a `response_type` or `response_mode` the client does not expect (`query` ↔ `fragment` ↔ `form_post` ↔ `web_message`) — so the code or token lands on an error page or a non-happy path that does not consume it. Then find a gadget on that page that leaks the URL: a weak `postMessage` listener, a third-party/analytics script that reads `location`, a URL rendered into a link or iframe. With `prompt=none` against a victim already signed in to the IdP, the chain needs one click.
- [ ] **nOAuth pattern:** the client keys accounts on a mutable or unverified claim (`email`, `preferred_username`, `upn`) instead of the stable subject (`sub`; Entra ID `oid` + `tid`). Set that claim to B's address in an IdP tenant you control.
- [ ] **IdP mix-up:** a client that supports several IdPs and does not check the issuer of the response (RFC 9207 `iss`) can be made to send an honest IdP's code to the attacker's IdP.
- [ ] As a provider: open dynamic client registration; `logo_uri`/`jwks_uri`/`sector_identifier_uri`/`request_uri` fetched server-side → SSRF (`vectors-server.md`).
- [ ] Refresh-token reuse detection: replay a rotated refresh token — does the server revoke the whole token family? Two refreshes with the same token in one single-packet group → two live families. Missing rotation or reuse detection alone is a hardening miss; report it with a source that hands the attacker the refresh token (XSS reading `localStorage`, a `Referer` or log leak, a mobile backup).

## SAML
Use the Burp **SAML Raider** extension.

**L1 — Basic**
- [ ] Signature stripping: remove the `<ds:Signature>` element.
- [ ] Assertion replay.

**L2 — Intermediate**
- [ ] Self-signed certificate accepted (SAML Raider "clone certificate" + re-sign).
- [ ] `Recipient`, `Audience`, `NotOnOrAfter`, and `InResponseTo` not checked.
- [ ] XXE in the SAML response (`vectors-injection.md`).

**L3 — Advanced**
- [ ] Signature wrapping (XSW1–XSW8 in SAML Raider).
- [ ] Comment injection in NameID: `b@x.com<!---->.attacker.com` — canonicalisation and text extraction disagree on the value.
- [ ] XSLT in a `<ds:Transform>` that the library evaluates before verifying the signature → OOB fetch to OAST.
- [ ] Library parser differentials: two XML parsers in one verification path (e.g. ruby-saml's REXML/Nokogiri split, CVE-2025-25291/25292; signature bypass CVE-2024-45409). Fingerprint the SP library and check the advisory's affected versions before testing.

## JWT
Use `jwt_tool` and the Burp **JWT Editor** extension.

**L1 — Basic**
- [ ] Signature not verified at all (change a claim, keep the old signature).
- [ ] `alg: none` (and `None`, `NONE`) with the signature removed.
- [ ] `exp` not enforced; token still valid after logout/password change.

**L2 — Intermediate**
- [ ] RS256 → HS256 confusion, signed with the server's public key (fetch from `/.well-known/jwks.json`).
- [ ] Weak HMAC secret: `hashcat -m 16500 jwt.txt wordlist.txt` (offline, against your own token).
- [ ] Embedded `jwk` header accepted.
- [ ] Claim tampering: `sub`, `role`, `org`, `email` after any of the bypasses above.

**L3 — Advanced**
- [ ] `kid` header: path traversal (`../../../../dev/null`, sign with an empty key); SQLi (`x' UNION SELECT 'k'-- -`, then sign with `k`).
- [ ] `jku` / `x5u` pointing at your own JWKS; if an allowlist applies, serve it through an open redirect or a file upload on an allowed host.
- [ ] Algorithm confusion with no published key: derive candidate RSA public keys from two server-signed tokens (`portswigger/sig2n`, `rsa_sign2n`) and try each as the HS256 secret.
- [ ] ECDSA "psychic signatures" (CVE-2022-21449): Java 15–18 before the April 2022 CPU accepted `r = s = 0` for ES256/384/512. Only when the stack fingerprints as an affected JVM.
- [ ] Cross-service acceptance: a token from another service (staging, mobile, a sister product, a lower-privileged client) is accepted because the services share a signing key and skip `aud`/`iss` checks.

## QR-code and scan-to-approve login

A browser shows a code; an already-authenticated app scans it and confirms; the browser becomes
logged in. Common on collaboration suites, messengers and consumer apps, and frequently built
in-house rather than on RFC 8628 — so the bindings that the device-code spec writes down are
often the ones missing here. The device authorization grant above is the standardised cousin;
run both lists where the product has both.

The flow has four values and the bugs are all in how they are bound: the **pre-auth session**
the browser holds, the **code** displayed, the **scan** that identifies the code to the app, and
the **confirm** that upgrades the session.

**L1 — Basic**
- [ ] Read what the code actually encodes — decode it, do not photograph it. A URL, an opaque id, a JWT, or a token already sufficient to authenticate? A code that *is* the credential is the finding.
- [ ] Poll the status endpoint for a code you did not generate: does it leak the scanning user's name, avatar, email or tenant before any confirmation? Pre-confirmation disclosure is its own finding.
- [ ] Does the code expire, and is it single-use? Re-scan an already-confirmed code; confirm twice.
- [ ] Is the confirm step authenticated **and** scoped to the scanner? Replay the confirm call with A's session against a code B's browser generated.

**L2 — Intermediate**
- [ ] **Browser binding.** The whole security of this flow is that only the browser that asked for the code gets the session. Request a code in browser 1, scan it with the app, then poll or complete in browser 2 — a different cookie jar, a different IP. If browser 2 is logged in, the pre-auth session is not bound to its requester and the attacker only needs the victim to scan a code the attacker generated.
- [ ] **Scope of the resulting session**: does the scan produce the same session as a password login, or a weaker one? If the same, does it bypass a step the password path enforces — MFA, device approval, a reauthentication prompt, a terms-acceptance gate?
- [ ] **CSRF on the confirm**, on the app side. If the confirm is an ordinary authenticated request with no unguessable value tied to the scan, a page opened in the app's own web view may be able to make it.
- [ ] **Downgrade and channel swap**: the same code id submitted to a sibling endpoint — another API version, the mobile API, a different regional host — where the binding check may not have been copied.
- [ ] Error-shape oracle: does an invalid, expired and already-used code answer differently? That is the enumeration surface for a short code.

**L3 — Advanced**
- [ ] **Time disagreement**: scan, then change state before confirming — revoke the scanning session, change the account's password, remove the user from the tenant. Does the confirm still mint a session for a now-invalid identity?
- [ ] **Race the confirm** against the code's expiry or its single-use marking: two confirms in one packet, or confirm as the code expires.
- [ ] **Cross-tenant or cross-product codes**: a code generated by one tenant's login page, or by a sibling product sharing the identity provider, accepted by another.
- [ ] Where the code is a deep link the app opens, the whole mobile deep-link surface applies — the scan may not need a camera at all (`mobile-app-security` → LINK position).

**Proof:** your own A and B. The honest severity question is what the victim had to do: a flow where the attacker must get the victim to scan an attacker-supplied code is phishing-shaped and many programs rate it low or exclude it, whereas a **missing browser binding** or a **replayable confirm** is a protocol flaw that stands on its own. Decide which you have before writing it up.

## Application-specific passwords and long-lived client credentials

Where a product supports a protocol client it cannot show a login page to — CalDAV, CardDAV,
IMAP/SMTP, a Git endpoint, an SDK — it usually issues an app password or a personal access
token. These are created once, live for years, and are routinely scoped more widely than the
UI implies.

**L1 — Basic**
- [ ] Enumerate what the product can issue and through which endpoint. Create one as A.
- [ ] **Scope**: the UI says "calendar". Try it against mail, files, contacts, the main API and the admin API. An app password that authenticates the whole account is the finding, and the UI's label is not the control.
- [ ] Does creating one require re-authentication? Does it require MFA, when the account has MFA? An app password that **bypasses MFA** is an MFA bypass, not a convenience.
- [ ] Is it shown again after creation, or recoverable from the UI or the API?

**L2 — Intermediate**
- [ ] **Revocation**: revoke it, then reuse it; change the account password, then reuse it; remove the user from the tenant, then reuse it. Each is a separate enforcement point and they are rarely all wired.
- [ ] Can A create one *for* B — a parameter naming the user, an admin endpoint with no role check (`Authorization — BFLA`)?
- [ ] Rate limiting and lockout on the protocol endpoint that accepts it: these endpoints often sit outside the web login's counters, which makes them the weak door for a credential the account also uses elsewhere. Policy first — brute force is excluded on many programs.
- [ ] Does the tenant admin have a policy to forbid them, and is it enforced server-side?

**Proof:** one authenticated call per surface you are testing the scope against, from A's own credential, and stop. Never a guessing campaign.

## Wallet sign-in (SIWE / EIP-4361)
**L1 — Basic**
- [ ] Nonce reuse: replay a captured signed message and signature after a successful login.
- [ ] `expiration-time` / `issued-at` not enforced: replay an expired message.

**L2 — Intermediate**
- [ ] Nonce not bound to the session that requested it: request a nonce in one session, sign it, redeem it in another.
- [ ] `domain` / `uri` not checked: a message signed for another site (the attacker's dApp) is accepted here.
- [ ] `chain-id` not checked: a message signed on another chain is accepted.

**L3 — Advanced**
- [ ] Address taken from the request body instead of recovered from the signature: sign with A's wallet, submit B's address.
- [ ] Address normalisation: checksum vs. lowercase forms map to two accounts, or one account matches both.
- [ ] Smart-contract wallets (EIP-1271): `isValidSignature` result not checked, or checked against the wrong contract or chain.

## Passkeys / WebAuthn
Chrome DevTools → **WebAuthn** panel gives a virtual authenticator for the normal flows. For the server-side checks, use a scripted software authenticator holding A's key: the key is yours, so you can sign any `clientDataJSON` and `authenticatorData` you build, and only the server's verification stands between that and a login.

**L1 — Basic**
- [ ] Add a passkey without re-authentication (fresh password or current factor) → with CSRF/XSS, a persistent login on B's account (same shape as *MFA* → change device).
- [ ] Remove a passkey, or the last factor, without re-authentication.
- [ ] Replay a captured, successful login assertion.

**L2 — Intermediate**
- [ ] Challenge: single use, short-lived, bound to the session that requested it — request it in one session, redeem the signed assertion in another.
- [ ] Signed fields not verified: `clientDataJSON.origin` (sign with another origin), `type` (`webauthn.create` vs `webauthn.get`), `rpIdHash` in `authenticatorData`, the user-present / user-verified flags when policy requires UV.
- [ ] Fallbacks around the passkey: password-only or magic-link login still accepted, or recovery replaces the passkey with a weaker factor (*MFA* → weaker fallbacks).

**L3 — Advanced**
- [ ] Credential not bound to the account: a valid assertion from A's credential submitted with B's username or `userHandle` — does the server check that the credential ID belongs to the user it logs in?
- [ ] Registration with a credential ID already registered to another account (a software authenticator lets you choose it); look for a lookup that then resolves to the wrong user. The spec says the RP should reject it.
- [ ] Attestation policy: when the program states only certain authenticators are allowed (hardware keys for admins), check whether `fmt: none` or self-attestation is accepted. Impact rests on that stated policy.
- [ ] `rpId` scope: an `rpId` of the registrable domain (`target.com`) lets any subdomain origin request assertions — if the server also accepts any subdomain origin, XSS or a takeover on a sibling subdomain logs in as the user.

---

## Authorization — third-party app platforms, scopes, and acting-as

A product with an "open platform", app directory, bot framework or integration marketplace has
a second authorization system beside its own: an **app** is installed into a **tenant** by an
**admin** who grants it **scopes**, and it then acts either as itself, as the tenant, or on a
**user's** behalf. Three identities, one set of endpoints. That is the condition for a
disagreement, and on a collaboration or SaaS product this surface is usually the highest-value
one in scope — it is a documented path into tenant data with a permission model as its only
guard.

Map it before testing: register a developer app of your own, read the platform's own
documentation for the credential types it issues and the scope names it defines, and record in
`map.md` which endpoint accepts which credential.

**L1 — Basic**
- [ ] Enumerate the **credential types** and keep them straight: a credential representing the app itself, one representing the app-in-a-tenant, and one representing a user who authorised it. Then cross them: present each to an endpoint documented for another. An endpoint that accepts the app's own credential where it should require a user's is a confused deputy by construction.
- [ ] **Granted versus enforced.** Install your app with the narrowest possible scope set, then call the endpoints the scopes you did *not* grant are documented to protect. The grant screen is a UI; the check is server-side or it is not there.
- [ ] Scope at install time versus at call time: request a scope, have it granted, then check whether removing it from the app's configuration actually stops the existing credential working.
- [ ] Does the app's credential respect the **installing tenant's** boundary? Call a tenant-scoped endpoint with an identifier belonging to a tenant that never installed your app.

**L2 — Intermediate**
- [ ] **Acting-as.** Where the platform lets an app operate on a user's behalf, test whether it can name a user who never authorised it, an admin, or a user in another tenant — a user id in the body, a header, or an impersonation parameter.
- [ ] **Scope inheritance through parents.** A scope granting access to a container (a folder, a space, a channel, a project) usually resolves children through a different code path. Reach a child the grant should not cover, and reach it through search, export and activity-feed endpoints as well as the direct one.
- [ ] **Event subscriptions and webhooks.** What events does the platform deliver, and are they filtered by the app's scopes at delivery time or only at subscription time? Subscribe broadly and see what arrives. Then the signature question: are the deliveries signed, is the signature verified by the receiver, and can a third party forge one (`vectors-logic-api.md` → Idempotency and replay).
- [ ] **The app's own secrets**: how a client secret is rotated, whether an old one keeps working, whether the secret is scoped to one tenant's installation or shared across all of them.
- [ ] **Admin-only platform actions** called with a non-admin user's credential: installing an app, changing its scopes, approving it for the tenant (`Authorization — BFLA`).
- [ ] Bot and app identities inside the product's own permission model: can a bot be added to a private space, read history from before it joined, or be made to relay a message it should not see?

**L3 — Advanced**
- [ ] **Time disagreement on the grant**: authorise, capture the credential, then uninstall the app or remove the granting admin — does the credential still work? Caches of granted scopes are a common place for this to fail.
- [ ] **Identity disagreement**: a request carrying both an app credential and a user credential, or an app credential plus a user id header. Which one wins, and is it the one the product documents?
- [ ] **Cross-tenant through the app**: one app installed in both your tenants, used to read across them. This is the cleanest cross-tenant proof a platform offers, and it uses only your own data.
- [ ] Marketplace-level surfaces: an app listing's metadata rendered to other tenants' admins (a stored-XSS sink with an admin audience), a redirect or callback URL the directory shows, or a published app's assets fetched server-side (`vectors-server.md` → SSRF).

**Proof and gates:** your own tenants, your own app, your own users. Publishing an app anywhere other tenants could install it needs operator approval (`SKILL.md` → Always operator-approved). Stop at one record from the second tenant, redacted.

## Authorization — IDOR / BOLA
Save A's response and B's response as transcripts and compare them with
`harness.py diff --a <A> --b <B>`: it reports the **leaf-key sets**, retyped fields and
changed values separately. A sha or a byte count answers `DIFFERS` for any two identities'
own records — that is what two different people's data looks like — so the question it
cannot answer is the only one being asked: *is a field present for one and absent for the
other.* On a forty-field response that one field is the finding.
Do this for every endpoint that takes an object ID, for every HTTP method, using the identity-object map in `objects.md`.

Method, per operation:
1. Controls: A reaches A's object; B reaches B's object. Save both.
2. Replay B's object with A's session.
3. Replay A's object with B's session — tenants and roles are often asymmetric.
4. Replay as anonymous where that makes sense.
5. Writes: use a reversible marker value, confirm the change from the owner's own view, then restore the original. Never use a delete as the first proof.
6. Record one tracker result per operation with the attack response and the owner-side verification. Then check `reporting.md` → Gate 2: how would an attacker learn B's ID?

**L1 — Basic**
- [ ] Read, update, and delete each object type (and its sub-resources: `/orders/{id}/invoice`, `/docs/{id}/versions`).
- [ ] Sequential IDs: numeric ±1.

**L2 — Intermediate**
- [ ] ID format tricks: a UUID leaked elsewhere, base64-decoded GraphQL global IDs (`Z2lkOi8v...` → `gid://app/User/123`), hashed IDs that are just `md5(int)`.
- [ ] Wrap the ID: `{"id":123}` → `{"id":[123]}`, `{"id":{"id":123}}`, `id=123&id=456`, `id[]=456`.
- [ ] Move the ID: from path to query/body or the reverse; add `user_id`/`account_id` to a request that does not normally carry it.
- [ ] Swap the method: `GET` works but `PUT` is unchecked; `POST` with `X-HTTP-Method-Override: DELETE`.
- [ ] Alternate paths: older API versions, mobile API hosts, export/download/PDF/print endpoints, websocket messages, and GraphQL `node(id:)` queries — these are often checked less carefully than the main endpoint.
- [ ] Indirect references: IDs inside a JWT, a signed URL that is not bound to the user, a pre-signed S3 URL with a long expiry.

**L3 — Advanced**
- [ ] **Second-order IDOR:** store B's ID in a field the app dereferences later in a background job (default billing account, report owner, export destination, webhook config). The write is checked; the async read is not.
- [ ] Path normalisation on object routes: `/api/orgs/<A>/../<B>/invoices`, `/api/users/me%2f..%2f<B>` — a proxy authorises on the prefix, the app routes the normalised path.
- [ ] Time-based UUIDs (v1 — the third group starts with `1`): IDs created near the same moment are predictable from one sample (`guidtool`).
- [ ] Confused-deputy paths: Slack/Jira/Zapier integrations, OAuth apps acting for the user, and LLM assistant tools that fetch objects by ID with service-level credentials.

## Authorization — BFLA / vertical privilege escalation
**L1 — Basic**
- [ ] Every admin/staff endpoint found in JS bundles or specs, called with a regular-user session.

**L2 — Intermediate**
- [ ] Org member vs. org admin: invite users, change roles, change billing, delete the org, read the audit log, manage SSO/SCIM, create API keys.
- [ ] Role change on your own membership via mass assignment (`"role":"owner"`).
- [ ] Feature gates: paid-tier features called from a free account (often low unless it bypasses a security control).

**L3 — Advanced**
- [ ] Privileged writes through side doors: bulk endpoints, CSV import with a `role` column, SCIM endpoints with a user token, a GraphQL mutation that the REST API forbids.
- [ ] Client-held privilege: a flag in the bundle or a signed-but-unverified blob (`isStaff`, `permissions`) that the server trusts on some endpoints.
- [ ] **Demotion and removal windows.** In one of your own orgs, the owner account lowers or removes a second account of yours (the member) while the member acts:
  - In flight: a long-running privileged job the member started (export, bulk edit, report generation). Does it finish with the old rights, and does its output still reach the member?
  - Cached rights: privileges copied into a JWT, the session, or a per-node cache at login. After the demotion, repeat one privileged request at `RL`; the time until it fails is the window. Minutes, or the whole token lifetime, is a finding; well under a second is normal propagation.
  - Same packet: the demotion and a privileged request sent together (single-packet, `vectors-logic-api.md` → Race conditions). The request succeeding after the demotion's response is the race.
  - Proof: timestamps of the demotion response and of each later success, on your own accounts.

## Authorization — multi-tenant
**L1 — Basic**
- [ ] Swap `org_id`/`workspace_id`/`tenant` (header, subdomain, path, or body) to B's org while authenticated as A.

**L2 — Intermediate**
- [ ] Invitation tokens: accept an invite meant for another email; reuse a revoked invite; invite into an org you are not an admin of.
- [ ] A user removed from an org keeps access through their existing session, API keys, or webhooks.
- [ ] Search, autocomplete, mention, and share dialogs that list users or objects from other tenants.

**L3 — Advanced**
- [ ] Conflicting tenant claims: path org vs. `X-Tenant`/`X-Org-Id` header vs. subdomain vs. JWT `org` claim. The authorisation layer can read one while the data layer reads another.
- [ ] Tenant-unaware caches: per-org responses cached by URL alone (`vectors-server.md` → cache deception/poisoning).
- [ ] Domain claiming for SSO/auto-join: verify a domain you do not own (TXT record checked on the wrong name, suffix match such as `customer.com.attacker.com`), then auto-join or force SSO on B's users.
- [ ] Invite race: accept an invite in the same packet as its revocation.
