# Findings Log (working notes)

Raw notes captured during testing. This gets cleaned up and turned into the Canva report later — it's not the deliverable itself.

Template per finding:

```
## [ID] Short title
- Category: (e.g. Information Disclosure / Missing Header / Outdated Component / Configuration)
- Risk Level: Low / Medium / High
- Where found: (URL / port / header name)
- What we saw: (raw evidence — command output, header value, etc.)
- Why it matters: (plain-English business impact)
- Remediation: (practical fix)
- Evidence file: evidence/<folder>/<file>
```

---

## Findings

## F-01 Full port scan returned all ports filtered (pending confirmation)
- Category: Network Perimeter / Configuration
- Risk Level: Informational — likely NOT a vulnerability (needs confirmation below)
- Where found: `nmap -sV -sC -Pn testphp.vulnweb.com` (top 1000 TCP ports)
- What we saw: All 1000 scanned ports reported "filtered (no-response)", including no confirmed open ports. Scan took 211s (slow for a responsive host — suggests retries from dropped packets). Host itself responded as up; rDNS resolves to an AWS EC2 instance (us-west-2).
- Why it matters: Port 80/443 must actually be open since the site is browsable — so this result is either (a) the security group silently drops raw scan probes while still serving legitimate HTTP traffic, a genuine defensive control worth noting positively, or (b) the scan traffic was rate-limited/dropped somewhere in the path and the result is inconclusive. Follow-up needed before writing this up either way.
- Remediation: N/A pending confirmation
- Evidence file: evidence/nmap/testphp_vulnweb_connectivity_investigation.txt

**Update 1:** Root `sudo nmap -sS -Pn -p 80,443` also returned both ports filtered — but fast (7.21s vs. 211s), so it's a consistent block, not packet loss noise. Ruled out: naive "just a slow/lossy scan" explanation.

**Update 2:** `curl -I http://testphp.vulnweb.com` from the same Kali VM also timed out (21s, "Could not connect to server") — so it's not Nmap-specific fingerprinting; even plain HTTP requests from this VM's IP can't reach the target at all right now.

**Conclusion (resolved):** Tested from a third, unrelated network (Claude's own browser tool) — both `http://` and `https://testphp.vulnweb.com` failed to load, while a control site (`example.com`) loaded instantly. Three independent vantage points (Kali Nmap, Kali curl, unrelated third-party browser) all fail on this one target while succeeding on everything else. **Root cause: the target site itself is currently down/unreachable**, not a VM, ISP, or scan-fingerprinting issue. Reclassifying F-01 as inconclusive/not usable — no valid port-state conclusion can be drawn from an unreachable target.

**Decision:** Switched target to `demo.testfire.net` (IBM Altoro Mutual demo bank), confirmed reachable. All findings from this point on apply to the new target. See updated README.md.

---

## F-02 Expired SSL/TLS certificate on port 443
- Category: Transport Security / Configuration
- Risk Level: Medium
- Where found: `https://demo.testfire.net:443` (Nmap `ssl-cert` script; confirmed by browser connection failure)
- What we saw: Port 443 is open and serving TLS (Apache Tomcat/Coyote) with a certificate for CN=demo.testfire.net, valid 2025-05-21 to 2026-06-21 — expired roughly 3 months before testing (2026-09-25). Loading the site over HTTPS in a real browser confirms this: Chrome blocks the connection outright with `NET::ERR_CERT_AUTHORITY_INVALID` ("Your connection is not private") rather than a normal page load. That specific error means the certificate chain doesn't lead to a trusted root CA (on top of being expired) — so the certificate has more than one problem, not just an expiry date that lapsed.
- Why it matters: On a banking-themed site, a broken padlock is especially damaging to visitor trust, and it trains users to click through/ignore certificate warnings — exactly the habit phishing and MITM attacks exploit. The identity-verification purpose of TLS is defeated even though the underlying encryption algorithm itself is fine.
- Remediation: Replace the certificate with one from a trusted CA (e.g. Let's Encrypt) and automate renewal (certbot cron) so it can't silently lapse again; enforce HTTPS + HSTS once fixed.
- Evidence file: evidence/nmap/nmap_scan.txt (expiry dates) and evidence/screenshots/https_cert_error.png (the actual browser error)

## F-03 Application server directly exposed on port 8080
- Category: Exposed Service / Attack Surface
- Risk Level: Medium
- Where found: `http://demo.testfire.net:8080`
- What we saw: The raw Apache Tomcat app server responds directly on 8080, serving the same content as port 80.
- Why it matters: Any hardening/WAF rules applied at the front-end on 80/443 can potentially be bypassed by hitting the app server directly on 8080. It's also an unnecessary extra entry point — one more open door than the application needs.
- Remediation: Firewall port 8080 to internal/localhost access only, so the app server is reachable exclusively through the hardened front-end.
- Evidence file: evidence/nmap/nmap_scan.txt

## F-04 Server software version disclosed via HTTP header
- Category: Information Disclosure
- Risk Level: Low
- Where found: `Server` header on ports 80/443/8080 → `Apache-Coyote/1.1`
- What we saw: The response banner discloses the exact application server technology in use.
- Why it matters: Doesn't grant access by itself, but gives an attacker a head start — they know precisely which platform's known vulnerabilities to research instead of having to guess.
- Remediation: Suppress/genericize the `Server` header (Tomcat: set the `server` attribute on the Connector in `server.xml`, or strip it at a reverse proxy).
- Evidence file: evidence/nmap/nmap_scan.txt
- *Corroborated by OWASP ZAP passive scan: "Server Leaks Version Information via Server HTTP Response Header Field" (Low).*

## F-05 Missing standard security headers
- Category: Missing Security Headers / Configuration
- Risk Level: Medium
- Where found: HTTP response headers on `http://demo.testfire.net` (`curl -I`)
- What we saw: None of these are present: `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`, `Referrer-Policy`.
- Why it matters:
  - No `X-Frame-Options` → the page can be embedded in an invisible iframe on an attacker's site for clickjacking (tricking a logged-in user into clicking something like "transfer funds" while thinking they're clicking something else) — especially relevant for a banking app.
  - No `Content-Security-Policy` → if a script ever gets injected anywhere on the site (XSS), nothing stops it from running or exfiltrating data.
  - No `X-Content-Type-Options: nosniff` → browsers may guess a file's type instead of trusting the declared one, which can be abused in content-type confusion attacks.
  - No `Strict-Transport-Security` → nothing forces browsers onto HTTPS, so the site defaults to (and stays on) unencrypted HTTP — compounds F-02.
  - No `Referrer-Policy` → full page URLs can leak to third parties via the Referer header on outbound links.
- Remediation: Add these headers at the web server/reverse-proxy level — a standard baseline config, not custom code.
- Evidence file: evidence/headers/curl_headers.txt
- *Corroborated and detailed by OWASP ZAP passive scan: "Content Security Policy (CSP) Header Not Set" (Medium), "Missing Anti-clickjacking Header" (Medium), "X-Content-Type-Options Header Missing" (Low).*

## F-06 Session cookie missing Secure and SameSite attributes
- Category: Session Management / Cookie Security
- Risk Level: High
- Where found: `Set-Cookie: JSESSIONID=...; Path=/; HttpOnly` on `http://demo.testfire.net`
- What we saw: The session cookie sets `HttpOnly` (good) but is missing `Secure` (would restrict it to HTTPS-only) and `SameSite` (restricts cross-site sending, mitigating CSRF).
- Why it matters: Combined with F-02 (HTTPS effectively broken due to the expired cert), this session cookie is realistically transmitted in plaintext for the whole session. Anyone positioned on the same network (public Wi-Fi, compromised router) could capture `JSESSIONID` and hijack the logged-in session outright — on a banking-themed app, that's a direct path to account takeover. Missing `SameSite` also increases CSRF exposure.
- Remediation: Set `Secure` (requires F-02 fixed first) and `SameSite=Lax`/`Strict` on the session cookie.
- Evidence file: evidence/headers/curl_headers.txt
- *Corroborated by OWASP ZAP passive scan: "Cookie without SameSite Attribute" (Low).*

## F-07 Absence of anti-CSRF tokens
- Category: Session Management / CSRF
- Risk Level: Medium
- Where found: OWASP ZAP passive scan — "Absence of Anti-CSRF Tokens" (Medium), flagged across login.jsp, feedback.jsp, subscribe.jsp and other forms
- What we saw: Forms on the site don't include an anti-CSRF token (a unique, unpredictable value tied to the user's session that proves a request really originated from the site's own form).
- Why it matters: Combined with F-06 (session cookie missing `SameSite`), there is no CSRF defense layer at all. An attacker could host a page that silently submits one of these forms — in a real banking app, something like a funds transfer — using the victim's active session, with no click or visible interaction needed beyond visiting the attacker's page while logged in.
- Remediation: Add a unique, server-validated anti-CSRF token to every state-changing form; set `SameSite=Strict`/`Lax` on the session cookie as a second layer (same fix as F-06).
- Evidence file: evidence/zap/ (ZAP alert detail)

## F-08 Suspicious HTML comments in page source
- Category: Information Disclosure
- Risk Level: Low / Informational
- Where found: OWASP ZAP passive scan — "Information Disclosure - Suspicious Comments" (Informational), flagged on most pages
- What we saw: ZAP detected HTML comments in the page source worth reviewing; it flags their presence but not the content — needs a manual look (View Page Source / Ctrl+U in browser, search for `<!--`) to see what they actually say.
- Why it matters: Developer comments left in production output sometimes leak internal file paths, usernames, TODO notes about known issues, or commented-out debug code — small details that add up during attacker reconnaissance.
- Remediation: Strip HTML comments from production output (most templating/build pipelines can do this automatically); review current comments for anything sensitive first.
- Evidence file: evidence/zap/ (also grab a screenshot of the actual comment text once you find it in page source)

## F-09 Exposed REST API documentation (Swagger UI), including undocumented-auth admin endpoints
- Category: Information Disclosure / Attack Surface / API Security
- Risk Level: High
- Where found: `/swagger/index.html` (spec served from `/swagger/properties.json`), linked directly from the site footer as "REST API"
- What we saw: A full Swagger/OpenAPI UI ("AltoroJ REST API", base path `/api`) is publicly reachable with no authentication. It documents the complete operation surface, grouped by category:
  - **Login:** `GET /login`, `POST /login`
  - **Account:** `GET /account`, `GET /account/{accountNo}`, `GET /account/{accountNo}/transactions`, `POST /account/{accountNo}/transactions`
  - **Transfer:** `POST /transfer` — "Transfer funds between accounts"
  - **Feedback:** `POST /feedback/submit`, `GET /feedback/{feedbackId}`
  - **Admin:** `POST /admin/addUser`, `POST /admin/changePassword` — "Add and change user details"
  - **Logout:** `GET /logout`
  - Plus request/response model schemas (`login`, `newUser`, `transfer`, `feedback`, `dates`, and more).
- Why it matters: This goes beyond generic doc exposure — it hands an unauthenticated visitor a complete roadmap of every sensitive operation the app supports, including **administrative user-management endpoints** (`/admin/addUser`, `/admin/changePassword`) and the funds-transfer endpoint, plus the exact data shape each expects. Even without calling them, publicly confirming these admin endpoints exist (and their parameter models) removes nearly all of the reconnaissance work an attacker would otherwise need to do, and specifically highlights account-takeover and unauthorized-admin-action as the highest-value targets. The `{accountNo}` path parameter pattern is also worth noting as a potential IDOR concern (unconfirmed — would need active testing to verify whether account numbers are sequential/guessable and properly access-controlled).
- **Scope note:** Documentation was viewed only — no endpoint listed above was called.
- Remediation: Restrict API documentation to internal/authenticated access only (VPN, auth-gated docs portal, or excluding it entirely from the public-facing deployment). Independently of the docs, admin endpoints should always enforce server-side authorization regardless of whether their existence is publicly known.
- Evidence file: evidence/screenshots/ (Swagger UI screenshot — already captured)

## F-10 Suspected SSRF risk in "Server Status Check" feature (unconfirmed — observation only)
- Category: Server-Side Request Forgery (SSRF) / Input Validation
- Risk Level: High (potential) — **not confirmed**; flagged from passive observation of client-side code only
- Where found: `/status_check.jsp`
- What we saw: The page's JavaScript takes a `HostName` value (default `'AltoroMutual'`, hardcoded in the form) and passes it as a query parameter to `util/serverStatusCheckService.jsp?HostName=...`, which appears to perform a server-side lookup/connection to that host and return its status. The parameter is attacker-controllable from the client side.
- Why it matters: If the backend doesn't validate or allow-list the `HostName` value, this is a classic SSRF pattern — the value could potentially be pointed at internal-only services or cloud metadata endpoints the server can reach but the public internet can't, using the server itself as a proxy. SSRF is one of the higher-impact web vulnerability classes when confirmed.
- **Scope note:** This was intentionally NOT tested. Confirming SSRF requires submitting crafted hostname values and observing server-side behavior — that's active testing/exploitation, outside this engagement's read-only, passive scope. Flagged here as a recommended focus area for a separately scoped, authorized penetration test rather than confirmed ourselves.
- Remediation (if confirmed): Validate/allow-list acceptable `HostName` values server-side; never let client input directly drive an outbound server-side request; restrict the server's outbound access to internal ranges and cloud metadata IPs by default.
- Evidence file: evidence/screenshots/status_check_page_source.txt
