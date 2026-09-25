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

**Update 2:** `curl -I http://testphp.vulnweb.com` from the same Kali VM also timed out (21s, "Could not connect to server") — so it's not Nmap-specific fingerprinting; even plain HTTP requests from this VM's IP can't reach the target at all right now. Two live hypotheses:
  (a) The target/AWS security group has rate-limited or temporarily blocked this VM's source IP after the earlier scanning activity (plausible — this is a heavily-scanned public target, likely has abuse protection).
  (b) The Kali VM's own network path is the problem, unrelated to the target.
  Next step: test outbound connectivity to an unrelated site, and re-test the target with more diagnostic detail, to tell (a) from (b) apart.
