# Vulnerability Assessment Report — demo.testfire.net (Altoro Mutual)

Read-only, passive security assessment performed as part of the Future Interns Cyber Security internship task.

## Website Tested
- **Target:** http://demo.testfire.net
- **Owner/Purpose:** "Altoro Mutual" (AltoroJ) — an intentionally-vulnerable demo banking application originally by IBM, now maintained by HCL Technologies as part of the AppScan product line. Publicly provided for security testing and training; no authorization request was needed. Confirmed directly by the site's own footer disclaimer: *"This site is not a real banking site... published for the sole purpose of demonstrating the effectiveness of HCL products in detecting web application vulnerabilities."* The app is also open source (github.com/AppSecDev/AltoroJ).
- **Note:** Testing originally targeted `testphp.vulnweb.com` (Acunetix's demo app), but it was confirmed unreachable from three independent networks during testing (see `findings.md` F-01) and the assessment was switched to this target instead.

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
Final designed report (created in Canva): `report/Task_01_Assement_report.pdf`

## Disclaimer
This assessment was conducted for educational purposes against a public target that explicitly permits security testing. No systems were harmed, and no exploitation, brute forcing, or denial-of-service activity was performed.
