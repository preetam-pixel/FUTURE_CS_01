# Vulnerability Assessment Report
## Altoro Mutual — demo.testfire.net

**Submitted by:** [Your Name], Cyber Security Intern
**Assessment type:** Read-only, passive external vulnerability assessment
**Date:** September 2026

*(This file is the source content for the Canva report — copy sections in as-is or lightly adapt wording per slide/page.)*

---

## 1. Executive Summary

For this task, I ran a read-only, passive security review of Altoro Mutual's public-facing website (demo.testfire.net) — a demo banking app IBM/HCL publish specifically for people to practice this kind of assessment on. This was my first hands-on vulnerability assessment, so I kept strictly to the passive side of things: no login attempts, no exploitation, no brute-forcing, nothing that could disrupt the site. Just observing what's publicly visible and figuring out what it means.

I ended up with **9 findings**: **3 High**, **4 Medium**, **1 Low**, and **1 Informational**. I didn't try to exploit or prove any of the High findings myself — two are configuration mistakes with a fairly direct fix (an expired certificate, an API that shares more than it should), and one is a pattern I noticed but deliberately didn't test further, since actually confirming it would mean probing the target, which is outside what this assessment was supposed to cover.

**My takeaway:** the site's biggest weak spots are around *session and login security* (an expired SSL certificate and a session cookie that isn't locked down) and it *shares more internal detail than it should* (a publicly viewable admin API map and an application server that's reachable directly). None of it needs a rebuild — it's almost all configuration fixes, which was actually reassuring to see as a beginner: most real-world issues aren't exotic, they're small things that got missed.

---

## 2. Scope & Methodology

**What I did:**
- Looked only at public-facing pages of the target site
- Ran passive reconnaissance: port/service exposure (Nmap), HTTP header analysis, an OWASP ZAP passive scan, and manual review of page source and linked resources

**What I deliberately did not do:**
- Attempt login or bypass authentication
- Exploit anything I found
- Brute-force anything
- Any load/stress or Denial-of-Service testing
- Anything that could disrupt or harm the site

**Tools used:** Nmap, curl, OWASP ZAP (passive scan only), browser DevTools.

**A note on the target:** I originally started testing a different public demo site, but partway through I found it was down — I confirmed this from three separate networks before concluding it wasn't a problem on my end. I switched to demo.testfire.net (IBM/HCL's "Altoro Mutual" demo bank, published for exactly this kind of practice) and continued from there with the same approach.

---

## 3. Risk Summary

| ID | Finding | Risk Level |
|----|---------|-----------|
| F-06 | Session cookie missing Secure/SameSite attributes | **High** |
| F-09 | Exposed API documentation, including admin endpoints | **High** |
| F-10 | Suspected SSRF pattern in status-check feature (unconfirmed) | **High (unconfirmed)** |
| F-02 | Expired SSL/TLS certificate | Medium |
| F-03 | Application server directly exposed on port 8080 | Medium |
| F-05 | Missing standard security headers (CSP, anti-clickjacking, etc.) | Medium |
| F-07 | No CSRF protection on forms | Medium |
| F-04 | Server software version disclosed | Low |
| F-08 | Suspicious HTML comments in page source | Informational |

---

## 4. Detailed Findings

### F-06 — Session cookie isn't locked down (High)
**What I found:** The session cookie is missing two standard protections (`Secure` and `SameSite`). It only has one of the three protections a session cookie should have.
**Why it matters:** Because the site's HTTPS is also broken (see F-02), this session ID realistically travels in plain, readable text every time the site is used. Anyone sharing a network with a user — public Wi-Fi being the obvious case — could potentially capture it and take over their logged-in session. On a banking-styled app, that's basically the equivalent of someone reading over a customer's shoulder as they type their password.
**Fix:** Add the missing cookie protections and fix the certificate (F-02) so encryption actually applies. Should be a small configuration change once it's prioritized.

### F-09 — Full API map, including admin functions, publicly viewable (High)
**What I found:** A complete technical map of the site's backend — basically every action it can perform — is published at a public web address with no login required. This includes account access, fund transfers, and **admin functions for adding users and changing passwords**.
**Why it matters:** Normally an attacker has to spend real time mapping out what a target application can do before trying anything. Here, that work is handed to them for free, with the admin functionality specifically labeled. It doesn't prove those admin functions are unprotected, but it tells an attacker exactly where to aim first.
**Fix:** Move this documentation behind a login, or take it out of the public-facing environment entirely — it belongs in an internal developer environment, not the live site.

### F-10 — Possible weakness in the "Server Status" feature (High, unconfirmed)
**What I found:** A feature on the site checks whether a given server is "up" by taking a hostname as input and checking it from the server side. I noticed the pattern but deliberately didn't test it further, since doing that would mean actively probing the target — which goes past the read-only scope I was working within.
**Why it matters:** This exact pattern is one of the more serious classes of web vulnerability when it turns out to be exploitable (it can let an attacker use the target's own server to reach internal systems that aren't supposed to be publicly reachable). I can't say for sure it's exploitable without testing it, and I intentionally didn't.
**Fix:** I'd recommend this specific feature be the focus of a properly scoped, authorized follow-up penetration test — a passive review like this one can flag the pattern, but can't confirm or rule it out on its own.

### F-02 — SSL certificate expired and untrusted (Medium)
**What I found:** The site's security certificate (the thing that makes a browser show a padlock) expired about three months before I ran this assessment. Loading the site over HTTPS in a real browser confirms it's worse than just expired — Chrome blocks the connection outright with `NET::ERR_CERT_AUTHORITY_INVALID`, meaning the certificate chain doesn't lead to a trusted authority either.
**Why it matters:** Visitors get a broken/blocked connection instead of a working secure one. For a banking-styled brand, that's a bad trust signal — and it also trains users to click through security warnings, which is exactly the habit phishing attacks rely on.
**Fix:** Replace the certificate with one from a trusted CA and set up automatic renewal so it doesn't get forgotten again.

### F-03 — Backend application server directly reachable (Medium)
**What I found:** The application server itself (not just the normal website address) answers directly on an extra network port.
**Why it matters:** Any protections normally sitting in front of the main website could potentially be routed around by going straight through this extra door.
**Fix:** Close public access to this port; the app server should only be reachable through the normal, protected front door.

### F-05 — Missing standard browser security protections (Medium)
**What I found:** The site doesn't send several standard instructions that tell browsers how to protect visitors — things like protection against the page being secretly embedded elsewhere, and against malicious scripts.
**Why it matters:** These are seatbelt-style protections — they don't stop every attack by themselves, but not having them removes a safety net that costs nothing to keep. On a banking-styled site, the "page can be secretly embedded elsewhere" gap stood out to me especially (it enables clickjacking, where a user thinks they're clicking one thing but are actually tricked into clicking something else).
**Fix:** A one-time configuration change on the web server — pretty standard practice for any modern site.

### F-07 — No protection against forged form submissions (Medium)
**What I found:** Forms on the site (login, feedback, etc.) don't include a hidden safety token that proves a submission really came from the site itself.
**Why it matters:** Combined with F-06, an attacker's page could potentially trick a logged-in visitor's browser into submitting one of these forms on their behalf — without the visitor clicking anything suspicious or even realizing it happened.
**Fix:** Add this safety token to forms, and fix F-06's cookie settings as a second layer of protection.

### F-04 — Server software version publicly disclosed (Low)
**What I found:** The site's response headers name the exact backend software in use.
**Why it matters:** Minor on its own, but it saves an attacker a reconnaissance step by telling them exactly what to go research.
**Fix:** Configure the server to stop announcing this detail.

### F-08 — Developer comments left in page source (Informational)
**What I found:** My ZAP scan flagged HTML comments across the site as worth reviewing, since comments left in by developers sometimes contain internal notes.
**Why it matters:** Low risk on its own, but worth a quick manual check in case anything sensitive got left behind.
**Fix:** Review and strip unnecessary comments from what's actually sent to visitors' browsers.

---

## 5. Supporting Evidence

*(In the PDF, this section shows the actual screenshots embedded inline — not just referenced by filename. When rebuilding in Canva, place the real images here, not placeholder boxes.)*

**Target site homepage** — demo.testfire.net, the site tested for this assessment. *(evidence/screenshots/homepage.png)*

**F-02 — HTTPS connection blocked.** Loading the site over HTTPS throws `NET::ERR_CERT_AUTHORITY_INVALID`: the certificate is expired and doesn't chain to a trusted authority. *(evidence/screenshots/https_cert_error.png)*

**F-09 — Swagger UI at /swagger/index.html.** Publicly viewable with no login, listing every endpoint including `POST /admin/addUser` and `POST /admin/changePassword`. *(evidence/screenshots/swagger_ui_endpoints.webp)*

Raw tool output (Nmap scans, HTTP headers, OWASP ZAP passive-scan alerts and crawl history) is documented in full in the accompanying GitHub repository's `evidence/` folder, referenced by finding ID throughout this report.

---

## 6. Remediation Roadmap

**Do first (quick, high-impact):**
- Renew the SSL certificate (F-02)
- Add the missing cookie protections (F-06)
- Add standard browser security headers (F-05)
- Restrict the extra exposed server port (F-03)

**Do next (still fairly straightforward):**
- Add anti-forgery tokens to forms (F-07)
- Move API documentation behind a login / out of the public environment (F-09)
- Stop the server from announcing its exact software version (F-04)

**Process / follow-up:**
- Set up automatic certificate renewal so F-02 can't quietly happen again
- Review and clean up developer comments in page source (F-08)
- Get a scoped, authorized penetration test done specifically on the status-check feature to confirm or rule out F-10

---

## 7. Conclusion

None of what I found here needs a rebuild — most of it is configuration changes a dev or ops team could realistically clear in a single sprint. If I had to pick where to start, I'd say the session cookie/certificate pair (F-06/F-02) and the exposed API map (F-09) matter most, since together they weaken the site's core promise of a secure login and remove an attacker's need to guess where the sensitive functionality even lives. I'd prioritize those first, then follow up with a properly scoped test on the status-check feature (F-10) to settle the one thing this passive review could flag but not confirm on its own.

This was a genuinely useful first assessment to run — it made clear how much can be learned about a site just by looking, without touching anything.
