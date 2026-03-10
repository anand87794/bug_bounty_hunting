# OWASP ZAP Cheatsheet

> **Tool:** OWASP ZAP (Zed Attack Proxy)  
> **Cost:** 100% Free & Open Source  
> **Best for:** Web app scanning, API testing, passive + active recon  
> **vs Burp Suite:** ZAP = free automation, Burp = manual precision

---

## Setup

```bash
# Download
https://www.zaproxy.org/download/

# Set browser proxy
Host: 127.0.0.1
Port: 8080

# Import ZAP CA cert into browser
ZAP → Tools → Options → Dynamic SSL Certificates → Save
Firefox → Settings → Certificates → Import → select zap_root_ca.cer
```

---

## ZAP Workflow (in order)

```
1. Set proxy       → configure browser to use 127.0.0.1:8080
2. Spider          → auto-discover all endpoints
3. Ajax Spider     → for JS-heavy / React / Angular apps
4. Browse manually → passive scan runs automatically
5. Active Scan     → inject payloads, find real vulns
6. Fuzzer          → custom wordlist attacks on params
7. Review Alerts   → High → Medium → Low → Informational
8. Export Report   → HTML/XML for documentation
```

---

## 01 · Spider / Crawler

```bash
# Standard Spider — discovers links, forms, assets
Sites → Right-click target → Spider

# Ajax Spider — for JS-rendered pages (React, Angular, Vue)
Tools → Ajax Spider → Start Scan

# What it finds:
- All linked pages and endpoints
- Hidden forms and parameters
- JS-loaded API endpoints (Ajax Spider only)
- Sitemap coverage for active scanning
```

---

## 02 · Passive Scanning

```
- Runs automatically as you browse or spider
- Zero extra requests sent to server — completely safe
- Checks for: missing headers, insecure cookies, info disclosure
- Enable HUD for real-time overlay while browsing:
  View → Show ZAP HUD
```

---

## 03 · Active Scanning

```bash
# Run active scan on full target
Sites → Right-click target → Active Scan → Start Scan

# Scan specific endpoint only
History tab → Right-click request → Active Scan

# What it tests:
SQLi, XSS (reflected/stored), SSRF, Path Traversal,
XXE, Command Injection, SSTI, Open Redirect,
Remote File Inclusion, LDAP Injection, Header Injection

# Note: Active scan = real requests with payloads
# Only use on targets you have permission to test!
```

---

## 04 · Alert Levels — What to Chase

```
🔴 HIGH        → Verify immediately. Direct exploitability.
                 Examples: SQLi, XSS with cookie theft, RCE
                 → Test manually in Burp → Write full PoC

🟠 MEDIUM      → Significant risk, context-dependent
                 Examples: CSRF, Clickjacking, Open Redirect
                 → Verify impact → include in report

🟡 LOW         → Minor issues, often chain with others
                 Examples: Missing headers, cookie flags
                 → Document, note chaining potential

ℹ️  INFO        → Not vulnerabilities — just observations
                 Examples: Server version, tech stack
                 → Use for recon, skip in final report
```

---

## 05 · Fuzzer

```bash
# Right-click any request in History tab → Fuzz
# Highlight the parameter value → Add → Payload

# Best payload sources:
/usr/share/seclists/Fuzzing/
/usr/share/seclists/Payloads/

# SQLi payloads
/usr/share/seclists/Fuzzing/SQLi/Generic-SQLi.txt

# XSS payloads
/usr/share/seclists/Fuzzing/XSS/XSS-Cheat-Sheet.txt

# Directory brute force
/usr/share/seclists/Discovery/Web-Content/common.txt

# After fuzzing:
Sort by Response Code → 200 = interesting
Sort by Response Size → anomalies = potential bug
Sort by Response Time → slow = potential SQLi/SSRF
```

---

## 06 · Essential Add-ons

```
Install via: Marketplace icon (puzzle piece) in ZAP toolbar

Must-have add-ons:
├── JWT Support          → decode + test JWT tokens
├── OpenAPI/Swagger      → auto-import & scan API specs
├── GraphQL Support      → enumerate + test GraphQL
├── Technology Detection → identify frameworks/CMS
├── Active Scan Rules    → extra vuln checks
├── Retire.js            → detect vulnerable JS libraries
└── FuzzDB               → extended payload lists
```

---

## Useful Keyboard Shortcuts

| Action | Shortcut |
|--------|----------|
| Open Spider | Ctrl+Alt+S |
| Open Fuzzer | Ctrl+Shift+F |
| Open Active Scanner | Ctrl+Alt+A |
| Show Alerts | Ctrl+Alt+B |
| Break (intercept) | Ctrl+B |
| Step (release one) | Ctrl+S |

---

## Generate Report

```bash
# In ZAP:
Report → Generate Report

# Formats:
HTML  → send to client / save for portfolio
XML   → parse programmatically
JSON  → integrate with CI/CD pipeline
MD    → GitHub / Notion documentation
```

---

## ZAP vs Burp Suite

| Feature | OWASP ZAP | Burp Suite |
|---------|-----------|------------|
| Cost | Free | $449/yr (Pro) |
| Active Scan | ✅ Auto | Manual (Pro) |
| Fuzzer | ✅ Built-in | Limited (free) |
| Repeater-style | ✅ Requester | ✅ Repeater |
| Intruder | ✅ Fuzzer | ✅ Intruder |
| Scanner | ✅ Auto | ✅ Pro only |
| API scanning | ✅ Add-on | ✅ Built-in |
| Best for | Automation | Manual testing |

---

## ZAP Checklist

```
☐  Set 127.0.0.1:8080 proxy + import CA cert
☐  Run Standard Spider on target
☐  Run Ajax Spider for JS-heavy pages
☐  Browse all features manually (passive scan)
☐  Run Active Scan on full target
☐  Review ALL High alerts — verify manually
☐  Use Fuzzer on login, search, API params
☐  Install JWT Support + OpenAPI add-ons
☐  Export HTML report for documentation
☐  Cross-check findings with Burp Suite
```

---

*Daily Bug Bounty notes — follow for more.*
