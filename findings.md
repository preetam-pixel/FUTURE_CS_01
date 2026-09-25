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
- Evidence file: evidence/nmap/nmap_scan.txt (copy your saved output here)

**Update 1:** Root `sudo nmap -sS -Pn -p 80,443` also returned both ports filtered — but fast (7.21s vs. 211s), so it's a consistent block, not packet loss noise. Ruled out: naive "just a slow/lossy scan" explanation.

**Update 2:** `curl -I http://testphp.vulnweb.com` from the same Kali VM also timed out (21s, "Could not connect to server") — so it's not Nmap-specific fingerprinting; even plain HTTP requests from this VM's IP can't reach the target at all right now.

**Conclusion (resolved):** Tested from a third, unrelated network (Claude's own browser tool) — both `http://` and `https://testphp.vulnweb.com` failed to load, while a control site (`example.com`) loaded instantly. Three independent vantage points (Kali Nmap, Kali curl, unrelated third-party browser) all fail on this one target while succeeding on everything else. **Root cause: the target site itself is currently down/unreachable**, not a VM, ISP, or scan-fingerprinting issue. Reclassifying F-01 as inconclusive/not usable — no valid port-state conclusion can be drawn from an unreachable target.

**Decision:** Switched target to `demo.testfire.net` (IBM Altoro Mutual demo bank), confirmed reachable. All findings from this point on apply to the new target. See updated README.md.

---

## F-02 Expired SSL/TLS certificate on port 443
- Category: Transport Security / Configuration
- Risk Level: Medium
- Where found: `https://demo.testfire.net:443` (Nmap `ssl-cert` script; confirmed by browser connection failure)
- What we saw: Port 443 is open and serving TLS (Apache Tomcat/Coyote) with a certificate for CN=demo.testfire.net, valid 2025-05-21 to 2026-06-21 — expired roughly 3 months before testing (2026-09-25). This is why HTTPS connections fail outright in-browser instead of showing a normal warning.
- Why it matters: On a banking-themed site, a broken padlock is especially damaging to visitor trust, and it trains users to click through/ignore certificate warnings — exactly the habit phishing and MITM attacks exploit. The identity-verification purpose of TLS is defeated even though the underlying encryption algorithm itself is fine.
- Remediation: Renew the certificate and automate renewal (e.g. Let's Encrypt + certbot cron) so it can't silently lapse again; enforce HTTPS + HSTS once renewed.
- Evidence file: evidence/nmap/nmap_scan.txt (ssl-cert script output shows the expiry dates)

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

## F-06 Session cookie missing Secure and SameSite attributes
- Category: Session Management / Cookie Security
- Risk Level: High
- Where found: `Set-Cookie: JSESSIONID=...; Path=/; HttpOnly` on `http://demo.testfire.net`
- What we saw: The session cookie sets `HttpOnly` (good) but is missing `Secure` (would restrict it to HTTPS-only) and `SameSite` (restricts cross-site sending, mitigating CSRF).
- Why it matters: Combined with F-02 (HTTPS effectively broken due to the expired cert), this session cookie is realistically transmitted in plaintext for the whole session. Anyone positioned on the same network (public Wi-Fi, compromised router) could capture `JSESSIONID` and hijack the logged-in session outright — on a banking-themed app, that's a direct path to account takeover. Missing `SameSite` also increases CSRF exposure.
- Remediation: Set `Secure` (requires F-02 fixed first) and `SameSite=Lax`/`Strict` on the session cookie.
- Evidence file: evidence/headers/curl_headers.txt
