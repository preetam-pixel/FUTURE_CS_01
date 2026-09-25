# Vulnerability Assessment Report — testphp.vulnweb.com

Read-only, passive security assessment performed as part of the Future Interns Cyber Security internship task.

## Website Tested
- **Target:** http://testphp.vulnweb.com
- **Owner/Purpose:** Acunetix intentionally-vulnerable demo application, publicly provided for security testing and training. No authorization request was needed as the site explicitly permits this use.

## Scope

**In scope (performed):**
- Passive/read-only reconnaissance of public-facing pages
- Nmap service/version scanning
- HTTP security header analysis
- OWASP ZAP passive scan
- Review of client-side exposure (cookies, outdated libraries, verbose error messages)

**Out of scope (not performed):**
- Login bypass or authentication attacks
- Exploitation of any identified vulnerability
- Brute force attacks
- Denial-of-Service (DoS) or any load/stress testing
- Any action that could degrade or harm the target site

## Tools Used
- **Nmap** — port & service exposure analysis
- **OWASP ZAP (Passive Scan only)** — automated passive vulnerability identification
- **Browser DevTools** — header, cookie, and client-side inspection

## Repository Structure
```
/README.md          — this file
/findings.md         — working notes log (raw findings before formatting into the report)
/evidence/           — screenshots and raw tool output supporting each finding
  /nmap/
  /headers/
  /zap/
  /screenshots/
/report/             — final PDF export of the Canva report
```

## Report
Final designed report: `report/vulnerability-assessment-report.pdf` *(added once complete)*

## Disclaimer
This assessment was conducted for educational purposes against a public target that explicitly permits security testing. No systems were harmed, and no exploitation, brute forcing, or denial-of-service activity was performed.
