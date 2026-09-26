# Vulnerability Assessment Report
## Altoro Mutual — demo.testfire.net

**Prepared for:** Altoro Mutual (demo/training environment)
**Assessment type:** Read-only, passive external vulnerability assessment
**Date:** September 2026

*(This file is the source content for the Canva report — copy sections in as-is or lightly adapt wording per slide/page.)*

---

## 1. Executive Summary

We performed a read-only, passive security review of Altoro Mutual's public-facing website (demo.testfire.net). No login attempts, exploitation, brute-forcing, or disruptive testing were carried out — this was a "look, don't touch" assessment, the same first step a real client engagement would start with.

We identified **9 findings**: **3 High**, **4 Medium**, **1 Low**, and **1 Informational**. None of the High findings were exploited or proven during this engagement — two are configuration issues with a clear, direct fix (expired certificate, an unusually exposed API), and one is a suspected-but-unconfirmed risk pattern flagged for a follow-up test.

**Bottom line:** the site's biggest gaps are around *trust and session security* (an expired SSL certificate and a session cookie that isn't locked down) and *oversharing of internal details* (an exposed admin API map and a directly-reachable application server). None of these require complex fixes — most are configuration changes, not a rebuild.

---

## 2. Scope & Methodology

**In scope:**
- Public-facing pages of the target site only
- Passive reconnaissance: port/service exposure (Nmap), HTTP header analysis, OWASP ZAP passive scan, manual review of page source and linked resources

**Explicitly out of scope (not performed):**
- Login attempts or authentication bypass
- Exploitation of any identified weakness
- Brute-force attacks
- Denial-of-Service or load/stress testing
- Any action that could disrupt or harm the site

**Tools used:** Nmap, curl, OWASP ZAP (passive scan only), browser DevTools.

**Note on target selection:** Testing originally targeted a different public demo site, which was confirmed down across three independent networks partway through the engagement. The assessment was redirected to demo.testfire.net (IBM/HCL's "Altoro Mutual" demo bank, explicitly published for this purpose) with no loss of methodology.

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
**What we found:** The session identifier cookie is missing two standard protections (`Secure` and `SameSite`). It only has one of the three protections a session cookie should have.
**Why it matters:** Because the site's HTTPS is also broken (see F-02), this session ID realistically travels in plain, readable text every time the site is used. Anyone sharing a network with a user — public Wi-Fi being the obvious case — could potentially capture it and take over their logged-in session. On a banking-styled app, that's the equivalent of someone reading over a customer's shoulder as they type their password.
**Fix:** Add the missing cookie protections and fix the certificate (F-02) so encryption actually applies. This is a configuration change, typically minutes of work once prioritized.

### F-09 — Full API map, including admin functions, publicly viewable (High)
**What we found:** A complete technical map of the site's backend — every action it can perform — is published at a public web address with no login required. This includes account access, fund transfers, and **admin functions for adding users and changing passwords**.
**Why it matters:** Normally, an attacker has to spend real time and effort mapping out what a target application can do before attempting anything. Here, that work is handed to them for free, with the admin functionality specifically called out. It doesn't prove those admin functions are unprotected, but it tells an attacker exactly where to aim.
**Fix:** Move this documentation behind a login, or take it out of the public-facing environment entirely — it belongs in an internal developer environment, not the live site.

### F-10 — Possible weakness in the "Server Status" feature (High, unconfirmed)
**What we found:** A feature on the site checks whether a given server is "up" by taking a hostname as input and checking it from the server side. We identified the pattern but deliberately did not test it further, since doing so would mean actively probing the target — outside the read-only scope of this engagement.
**Why it matters:** This exact pattern is one of the more serious classes of web vulnerability when it's exploitable (it can let an attacker use the target's own server to reach internal systems that aren't supposed to be publicly reachable). We can't confirm it's exploitable without testing it, which we didn't do.
**Fix:** We recommend this specific feature be the focus of a properly scoped, authorized follow-up penetration test, since a passive review can flag the pattern but can't confirm or rule it out.

### F-02 — SSL certificate expired (Medium)
**What we found:** The site's security certificate (the thing that makes a browser show a padlock) expired about three months before this assessment.
**Why it matters:** Visitors get a broken/blocked connection instead of a working secure one. For a banking-styled brand, that's a serious trust signal to get wrong — and worse, it trains users to click through security warnings, a habit that phishing attacks rely on.
**Fix:** Renew the certificate and set up automatic renewal so it can't be forgotten again.

### F-03 — Backend application server directly reachable (Medium)
**What we found:** The application server itself (not just the normal website address) answers directly on an extra network port.
**Why it matters:** Any protections normally sitting in front of the main website could potentially be routed around by going straight to this extra door.
**Fix:** Close public access to this port; the app server should only be reachable through the normal, protected front door.

### F-05 — Missing standard browser security protections (Medium)
**What we found:** The site doesn't send several standard instructions that tell browsers how to protect visitors (protections against having the page secretly embedded elsewhere, against malicious scripts, and a few others).
**Why it matters:** These are seatbelt-style protections — they don't stop every attack on their own, but their absence removes a safety net that costs nothing to have. On a banking-styled site, the "page can be secretly embedded elsewhere" gap is particularly relevant (it enables a trick called clickjacking, where a user thinks they're clicking one thing but are actually tricked into clicking something else).
**Fix:** A one-time configuration change on the web server, standard practice for any modern site.

### F-07 — No protection against forged form submissions (Medium)
**What we found:** Forms on the site (login, feedback, etc.) don't include a hidden safety token that proves a submission really came from the site itself.
**Why it matters:** Combined with F-06, an attacker's page could potentially trick a logged-in visitor's browser into submitting a form on their behalf — without the visitor clicking anything suspicious or even realizing it happened.
**Fix:** Add this safety token to forms, and fix F-06's cookie settings as a second layer of protection.

### F-04 — Server software version publicly disclosed (Low)
**What we found:** The site's response headers name the exact backend software in use.
**Why it matters:** Minor on its own, but it saves an attacker a reconnaissance step by telling them exactly what to research.
**Fix:** Configure the server to stop announcing this detail.

### F-08 — Developer comments left in page source (Informational)
**What we found:** Automated scanning flagged HTML comments across the site worth a manual look, since some comments left in by developers can contain internal notes.
**Why it matters:** Low risk by itself, but worth a five-minute manual check in case anything sensitive was left behind.
**Fix:** Review and strip unnecessary comments from what's actually sent to visitors' browsers.

---

## 5. Remediation Roadmap

**Do first (quick, high-impact):**
- Renew the SSL certificate (F-02)
- Add the missing cookie protections (F-06)
- Add standard browser security headers (F-05)
- Restrict the extra exposed server port (F-03)

**Do next (still straightforward):**
- Add anti-forgery tokens to forms (F-07)
- Move API documentation behind a login / out of the public environment (F-09)
- Stop the server from announcing its exact software version (F-04)

**Process / follow-up:**
- Set up automatic certificate renewal so F-02 can't recur
- Review and clean up developer comments in page source (F-08)
- Commission a scoped, authorized penetration test specifically targeting the status-check feature to confirm or rule out F-10

---

## 6. Conclusion

None of the issues found here require a rebuild — the majority are configuration changes a development or ops team could realistically clear in a single sprint. The two items worth the most attention are the session cookie/certificate combination (F-06/F-02), because together they weaken the site's core promise of a secure login, and the exposed API map (F-09), because it removes an attacker's need to guess where the sensitive functionality lives. We recommend those four be prioritized first, followed by a scoped follow-up test on the status-check feature (F-10) to close out the one item this passive review could flag but not confirm.
