# Bug Bounty Recon Workflow — 6-Step Process

> **Category:** Bug Bounty Methodology  
> **Week:** 4 — CTF & Career  
> **Best For:** Structured approach to any new target  
> **Rule:** Recon is 80% of hacking. Never skip it.

---

## The 6-Step Workflow

```
Step 01 → Target Selection
Step 02 → Passive Recon
Step 03 → Active Enumeration
Step 04 → Vulnerability Discovery
Step 05 → Exploitation & Proof
Step 06 → Report & Follow Up
```

---

## Step 01 — Target Selection

```bash
# Where to find good programs
hackerone.com/programs   → filter: bounty + new programs
bugcrowd.com/programs    → filter: reward + recently launched
intigriti.com            → European programs, less competition
yeswehack.com            → EU programs, growing

# What to look for:
✅ Wide scope (*.target.com — not just www.target.com)
✅ New programs (fewer hunters = more low-hanging fruit)
✅ Programs with assets that include APIs + mobile
✅ VDPs going private soon (read announcements)
✅ Recent acquisitions (new subdomain = new attack surface)

# Read the scope CAREFULLY:
- What's explicitly IN scope?
- What's explicitly OUT of scope?
- Any special rules? (no automated scanning, etc.)
- What's the payout structure?
```

---

## Step 02 — Passive Recon

```bash
# Subdomain discovery (passive — zero touch)
subfinder -d target.com -silent -o subs.txt
amass enum -passive -d target.com -silent >> subs.txt
curl -s "https://crt.sh/?q=%25.target.com&output=json" \
  | jq '.[].name_value' | tr -d '"' | sort -u >> subs.txt
sort -u subs.txt -o subs.txt

# Check which are live
cat subs.txt | httpx -silent -status-code -title -o live.txt

# Google Dorks — find leaked files
site:target.com ext:env OR ext:sql OR ext:bak
site:target.com inurl:admin OR inurl:login OR inurl:api
site:target.com intitle:"index of"

# GitHub secrets
site:github.com target.com password
site:github.com target.com api_key
site:github.com target.com DB_PASSWORD
trufflehog github --org=targetorg

# Shodan
org:"Target Company" http.title:admin
hostname:target.com port:8080,8443,3000

# Wayback Machine — find old endpoints
curl "https://web.archive.org/cdx/search/cdx?url=*.target.com/*&output=text&fl=original&collapse=urlkey"
```

---

## Step 03 — Active Enumeration

```bash
# Port scanning
nmap -sV -p- --open target.com -oN nmap_full.txt
nmap -sV -p80,443,8080,8443,3000,8000 target.com

# Directory brute force
gobuster dir -u https://target.com -w /opt/SecLists/Discovery/Web-Content/big.txt \
  -x php,html,js,txt,bak,old,zip -t 30 -o gobuster.txt

# Recursive with feroxbuster
feroxbuster -u https://target.com -w /opt/SecLists/Discovery/Web-Content/raft-large-dirs.txt \
  --filter-status 404 -d 3

# Tech stack fingerprinting
whatweb target.com -v
wafw00f target.com
curl -I https://target.com | grep -iE "server|x-powered-by|x-generator"

# API endpoint discovery
gobuster dir -u https://target.com/api -w /opt/SecLists/Discovery/Web-Content/api/objects.txt
ffuf -u https://target.com/FUZZ -w api-routes.txt -mc 200,201,401,403

# Check for Swagger/API docs
# /swagger  /api-docs  /openapi.json  /v1/docs  /graphql
```

---

## Step 04 — Vulnerability Discovery

```bash
# Authentication testing
curl -X GET https://target.com/api/v1/admin/users              # no auth
curl -H "Authorization: Bearer EXPIRED_TOKEN" https://target.com/api/me
curl -H "Authorization: Bearer ACCOUNT_B_TOKEN" https://target.com/api/account/A

# IDOR testing
GET /api/v1/invoices/1001  # your ID is 1002
GET /api/v1/orders/999
POST /api/create  {"user_id": 9999}

# Input fuzzing
sqlmap -u "https://target.com/search?q=test" --level=3 --risk=2
# XSS payloads on every input
# SSTI: {{7*7}}, ${7*7}, <%= 7*7 %>
# Path traversal: ../../../etc/passwd

# Mass assignment
POST /api/register
{"email":"x@x.com","password":"Test1","role":"admin","is_admin":true}

# Business logic
# Rate limiting: send 100 requests to /login — does it 429?
# Race condition: parallel requests to /redeem-coupon
```

---

## Step 05 — Exploitation & Proof

```bash
# Clean PoC documentation:
1. Screenshot the original request (your normal account)
2. Screenshot the modified request (changed ID/header)
3. Screenshot the response showing the bug
4. Note the exact change that caused the bug

# Never:
- Access real user data beyond confirming the bug
- Modify or delete anything
- Access admin functions beyond confirming access

# Impact assessment:
- Authentication required? (if no → Critical)
- Data exposed? (PII → High/Critical)
- Can it be chained? (SSRF + IDOR = bigger impact)
- CVSS score estimate?
```

---

## Step 06 — Report & Follow Up

```markdown
# Perfect report template:

**Title:** [Endpoint] — [Vuln Type] leads to [Impact]
Example: GET /api/v1/invoices/{id} — IDOR leads to unauthorized access to all user invoices

**Severity:** High / Critical

**Description:**
A brief explanation of what the vulnerability is and why it matters.

**Steps to Reproduce:**
1. Create Account A (attacker) — note user_id = 1002
2. Create Account B (victim) — note user_id = 1001
3. Log in as Account A
4. Send request: GET /api/v1/invoices/1001
   Authorization: Bearer [Account A token]
5. Observe: Response returns Account B's invoice data

**Proof of Concept:**
[Screenshot of request + response]

**Impact:**
Any authenticated user can access any other user's invoice data by
changing the ID parameter. This exposes financial records, PII, and
transaction history of all users on the platform.

**Remediation:**
Implement proper authorization checks — verify that the requesting
user owns the resource being accessed before returning data.

**CVSS Score:** 8.1 (High)
```

---

## Full Recon Script

```bash
#!/bin/bash
TARGET=$1
echo "[*] Starting recon on $TARGET"

# Passive
echo "[*] Subdomain enum..."
subfinder -d $TARGET -silent -o subs.txt
amass enum -passive -d $TARGET -silent >> subs.txt
sort -u subs.txt -o subs.txt

# Live check
echo "[*] Checking live hosts..."
cat subs.txt | httpx -silent -status-code -title -o live.txt

# Active
echo "[*] Dir brute force on main domain..."
gobuster dir -u https://$TARGET -w /opt/SecLists/Discovery/Web-Content/big.txt \
  -x php,html,js -t 30 -q -o gobuster.txt

# Tech
echo "[*] Tech fingerprint..."
whatweb https://$TARGET

echo "[+] Done! Check: subs.txt  live.txt  gobuster.txt"
```

---

## Checklist

```
☐  Read full scope — know what's in/out before starting
☐  Passive recon ALWAYS — subfinder + amass + crt.sh first
☐  Check GitHub early — leaked secrets = instant Critical
☐  Create 2 test accounts — needed for IDOR/auth testing
☐  Document everything — screenshot every step you take
☐  Submit early, fix later — establish timestamp first
☐  Follow up after 7 days if no triage response
☐  Write report in program's preferred format
```

---

*Daily Bug Bounty notes — follow for more.*
