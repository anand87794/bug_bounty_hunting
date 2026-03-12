# Nikto — Web Vulnerability Scanner

> **Category:** Recon & Vulnerability Scanning  
> **Type:** Free, Open Source  
> **Best For:** Quick recon, misconfiguration detection, header checks  
> **Bounty Value:** Low-Medium findings (chains to High with other bugs)

---

## What is Nikto?

Nikto is an open-source web server scanner that performs comprehensive tests against web servers — checking for dangerous files, outdated software, misconfigurations, and missing security headers. It's the first tool to run on any new target.

```bash
# Install
apt install nikto          # Kali/Debian
brew install nikto         # macOS
git clone https://github.com/sullo/nikto  # Manual
```

---

## 6 Scan Types

### 01 · Basic Scan & Target Discovery
```bash
# Basic scan
nikto -h target.com

# Custom port
nikto -h target.com -port 8080

# HTTPS target
nikto -h target.com -ssl

# Full HTTPS with port
nikto -h target.com -ssl -port 443

# Multiple targets from file
nikto -h targets.txt

# Sample output:
# + Server: Apache/2.4.49
# + X-Frame-Options: not present
# + OSVDB-3268: /admin/: Directory indexing enabled
```

### 02 · Understanding Nikto Output
```
+ prefix      = finding (vulnerability, misconfiguration, info)
- prefix      = informational only
OSVDB-XXXXX   = Open Source Vulnerability Database reference
               → Google "OSVDB-XXXXX" for full exploit details

Key findings to prioritize:
+ /admin/           → admin panel exposed
+ /backup/          → backup files accessible
+ /.git/            → source code exposed
+ /phpinfo.php      → PHP config exposed
+ outdated version  → check CVEs for that version
+ missing headers   → X-Frame, CSP, HSTS
```

### 03 · Scan with Authentication
```bash
# Basic HTTP authentication
nikto -h target.com -id admin:admin
nikto -h target.com -id admin:password
nikto -h target.com -id user:user123

# Use session cookie (grab from Burp)
nikto -h target.com -C "PHPSESSID=abc123; auth=xyz"

# Spoof user agent (bypass basic WAF)
nikto -h target.com -useragent "Mozilla/5.0 (Windows NT 10.0)"

# Use proxy (route through Burp Suite)
nikto -h target.com -useproxy http://127.0.0.1:8080
```

### 04 · Evasion & WAF Bypass Modes
```bash
# Evasion technique options:
nikto -h target.com -evasion 1   # Random URI encoding
nikto -h target.com -evasion 2   # Directory self-reference (./)
nikto -h target.com -evasion 3   # Premature URL ending
nikto -h target.com -evasion 4   # Prepend long random string
nikto -h target.com -evasion 5   # Fake parameter
nikto -h target.com -evasion 6   # TAB as request spacer
nikto -h target.com -evasion 7   # Random case sensitivity
nikto -h target.com -evasion 8   # Use Windows dir separator \

# Combine multiple evasions
nikto -h target.com -evasion 1478
```

### 05 · Save & Format Output
```bash
# Save as HTML (best for screenshots in reports)
nikto -h target.com -o report.html -Format htm

# Save as XML
nikto -h target.com -o report.xml -Format xml

# Save as CSV
nikto -h target.com -o report.csv -Format csv

# Save as plain text
nikto -h target.com -o report.txt -Format txt

# Tip: HTML output = professional bug bounty screenshots
```

### 06 · Scan Tuning & Combine with Tools
```bash
# Tuning options (scan only specific categories):
nikto -h target.com -Tuning 1   # Interesting files
nikto -h target.com -Tuning 2   # Misconfiguration
nikto -h target.com -Tuning 3   # Information disclosure
nikto -h target.com -Tuning 6   # Denial of service checks
nikto -h target.com -Tuning 9   # SQL injection
nikto -h target.com -Tuning x   # All checks (default)

# Time-limited scan
nikto -h target.com -maxtime 60s

# Pipe nmap output into Nikto
nmap -p80,443,8080,8443 --open -oG - target.com | nikto -h -

# Scan entire subnet
for ip in 192.168.1.{1..254}; do nikto -h $ip -maxtime 30s; done
```

---

## What Nikto Finds

| Finding | Severity | Example |
|---------|----------|---------|
| Default credentials | Critical | admin:admin works |
| Directory listing | Medium | /backup/ exposes files |
| Outdated software | High | Apache 2.4.49 (CVE-2021-41773) |
| Debug pages | High | /phpinfo.php, /server-status |
| Missing headers | Low | No CSP, no HSTS |
| Risky files | Medium | /admin/config.php, /.env |
| Dangerous methods | High | PUT/DELETE enabled |

---

## Full Recon Workflow

```bash
# Step 1 — Quick basic scan
nikto -h target.com -ssl -port 443

# Step 2 — Authenticated scan (if you have creds)
nikto -h target.com -ssl -id admin:admin -o auth_scan.html -Format htm

# Step 3 — Tune for specific findings
nikto -h target.com -Tuning 123 -o findings.html -Format htm

# Step 4 — If WAF detected, try evasion
nikto -h target.com -evasion 147 -ssl

# Step 5 — Export and review
# Open report.html → screenshot findings → add to report
```

---

## Checklist

```
☐  Always run: nikto -h target.com -ssl -port 443
☐  Check OSVDB references — Google each ID for exploit details
☐  Look for: /admin/, /.git/, /backup/, /phpinfo.php in output
☐  Save output as HTML for clean bug bounty report screenshots
☐  Re-run with -evasion if WAF blocks your scan
☐  Combine with gobuster — Nikto misses many hidden paths
☐  Route through Burp proxy to capture all Nikto requests
☐  Run authenticated scan if you have any valid credentials
```

---

## Nikto vs Other Scanners

| Feature | Nikto | Burp Suite | OWASP ZAP |
|---------|-------|-----------|-----------|
| Cost | Free | Free/Paid | Free |
| Speed | Fast | Slow | Medium |
| Active scan | Basic | Deep | Deep |
| Report quality | Basic | Pro | Good |
| Best for | Quick recon | Deep testing | Automation |

**Pro tip:** Run Nikto first for quick wins, then Burp Suite for deep manual testing.

---

## Report Template

```
Title: Directory Listing Enabled — /backup/ Exposes Database Dump

Severity: High

Found via: Nikto scan → OSVDB-3268

Steps to Reproduce:
1. Run: nikto -h target.com
2. Output: + OSVDB-3268: /backup/: Directory indexing found
3. Navigate to https://target.com/backup/
4. Observe: db_backup_2024.sql available for download
5. Download file — contains full user database with hashed passwords

Impact:
Full database exposure — all user emails, hashed passwords,
PII data accessible to any unauthenticated attacker.

Remediation:
- Disable directory listing in web server config
- Move backups outside web root
- Restrict access to /backup/ with authentication
```

---

## Resources

- [Nikto GitHub](https://github.com/sullo/nikto)
- [OSVDB References](https://github.com/sullo/nikto/wiki)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)

---

*Daily Bug Bounty notes — follow for more.*
