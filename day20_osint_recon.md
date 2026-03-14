# OSINT & Recon-ng — Web App Reconnaissance

> **Category:** Passive & Active Recon  
> **Tools:** Subfinder, Amass, theHarvester, Recon-ng, Shodan, Maltego  
> **Best For:** Subdomain discovery, email harvesting, tech fingerprinting, secret leaks  
> **Bounty Value:** Recon directly leads to High/Critical findings

---

## Why OSINT Matters

The best hackers spend 80% of their time on recon. Every subdomain, email, GitHub repo, and exposed service discovered passively = more attack surface = more bounties. OSINT means zero noise, zero detection, zero requests to target.

---

## 6 OSINT Techniques

### 01 · Passive Recon — No Touch Needed

```bash
# Google Dorks
site:target.com filetype:pdf
site:target.com inurl:admin
site:target.com ext:env OR ext:sql OR ext:bak OR ext:config
site:target.com intitle:"index of"
site:target.com "API_KEY" OR "password" OR "secret"
"@target.com" filetype:xls OR filetype:csv   # employee emails in spreadsheets

# Shodan
org:"Target Company"
hostname:target.com port:8080
ssl.cert.subject.cn:target.com http.title:admin
net:TARGET_IP_RANGE port:22,3306,5432,6379

# Wayback Machine
curl "https://web.archive.org/cdx/search/cdx?url=*.target.com/*&output=text&fl=original&collapse=urlkey"
# Finds all historically indexed URLs including deleted endpoints
```

### 02 · Subdomain Enumeration

```bash
# Subfinder (fastest, passive)
subfinder -d target.com -o subs.txt
subfinder -d target.com -silent | httpx -silent  # check which are live

# Amass (most comprehensive)
amass enum -passive -d target.com
amass enum -active -d target.com -brute         # active brute force
amass enum -d target.com -src                   # show sources

# Certificate Transparency (crt.sh)
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq '.[].name_value' | sort -u

# DNSx — resolve and filter
cat subs.txt | dnsx -silent -a -resp

# Combine everything
subfinder -d target.com -silent > subs.txt
amass enum -passive -d target.com -silent >> subs.txt
sort -u subs.txt | httpx -silent -o live_subs.txt
```

### 03 · Email & Employee OSINT

```bash
# theHarvester
theHarvester -d target.com -b google
theHarvester -d target.com -b linkedin
theHarvester -d target.com -b hunter
theHarvester -d target.com -b all -l 500 -f report.html

# hunter.io (web)
# https://hunter.io/domain-search → enter domain → all emails

# LinkedIn enumeration
# Search: site:linkedin.com "target company" "engineer" OR "developer"
# Note: names → guess email format: first.last@target.com

# Check email format
# https://emailformat.hunter.io/target.com

# Cross-check with breach databases
# https://haveibeenpwned.com/DomainSearch (requires subscription)
# https://dehashed.com
```

### 04 · Recon-ng Framework

```bash
# Start recon-ng
recon-ng

# Create workspace
workspaces create target_recon

# Install all modules
marketplace install all

# Key modules:
# Subdomain discovery
modules load recon/domains-hosts/hackertarget
options set SOURCE target.com
run

# Certificate transparency
modules load recon/domains-hosts/certificate_transparency
options set SOURCE target.com
run

# Email harvesting
modules load recon/domains-contacts/pgp_search
options set SOURCE target.com
run

# Shodan integration
modules load recon/hosts-ports/shodan_ip
keys add shodan_api YOUR_API_KEY
run

# View all data
show hosts
show contacts
show ports

# Export report
modules load reporting/html
options set FILENAME /tmp/recon_report.html
run
```

### 05 · Tech Stack Fingerprinting

```bash
# whatweb — command line
whatweb target.com
whatweb -v target.com           # verbose
whatweb target.com -a 3         # aggressive scan

# Output: WordPress[5.8], PHP[7.4.3], Apache[2.4.49], jQuery[3.5.1]
# → Search: "Apache 2.4.49 CVE" → CVE-2021-41773 Path Traversal!

# wafw00f — WAF detection
wafw00f target.com
# Know the WAF = know how to bypass it

# Browser tools
# Wappalyzer extension — instant on any page
# builtwith.com — full tech history

# HTTP headers fingerprinting
curl -I target.com | grep -iE "server|x-powered-by|x-generator|x-runtime"
# Server: Apache/2.4.49  →  check CVEs immediately
```

### 06 · GitHub & Code OSINT

```bash
# GitHub search (web)
# Search these patterns on github.com:
"target.com" password
"target.com" api_key
"target.com" secret
"target.com" aws_access_key
"target.com" DB_PASSWORD
org:targetorg filename:.env
org:targetorg filename:config.php

# truffleHog — automated secret scanning
trufflehog github --org=targetorg
trufflehog git https://github.com/target/repo

# gitrob — scan org for sensitive files
gitrob analyze targetorg

# gitleaks
gitleaks detect --source . --report-path report.json

# GitDorker
python3 gitdorker.py -tf tokens.txt -q target.com -d dorks.txt
```

---

## Full OSINT Workflow

```bash
# Phase 1 — Passive (15 mins, zero noise)
subfinder -d target.com -silent -o subs.txt
amass enum -passive -d target.com -silent >> subs.txt
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq '.[].name_value' >> subs.txt
sort -u subs.txt -o subs.txt

# Phase 2 — Live check
cat subs.txt | httpx -silent -status-code -title -o live.txt

# Phase 3 — Email harvest
theHarvester -d target.com -b all -l 500

# Phase 4 — Tech fingerprint
cat live.txt | awk '{print $1}' | whatweb -i -

# Phase 5 — GitHub secrets
trufflehog github --org=targetorg

# Phase 6 — Shodan
shodan search "org:\"Target Inc\"" --fields ip_str,port,transport,hostnames
```

---

## Google Dork Cheatsheet

```
site:target.com                           # all indexed pages
site:target.com inurl:admin               # admin panels
site:target.com filetype:pdf              # PDF documents
site:target.com ext:env                   # .env files
site:target.com ext:sql                   # SQL dumps
site:target.com intitle:"index of"        # directory listings
site:target.com intext:"password"         # password mentions
site:target.com "DB_PASSWORD"             # env variable names
"@target.com" filetype:xls               # email lists
inurl:target.com/api swagger              # API docs
cache:target.com                          # cached version
related:target.com                        # similar sites
```

---

## Checklist

```
☐  Run subfinder + amass — find ALL subdomains first
☐  Check crt.sh for certificate transparency logs
☐  Search GitHub for target.com + password/secret/api_key
☐  Run theHarvester for emails — check breach databases
☐  Shodan: org:"Company" — find exposed services/panels
☐  Wayback Machine — find old endpoints and hidden paths
☐  Fingerprint tech stack — search CVEs for versions found
☐  Check Wayback for deleted /api/ endpoints still live
```

---

## High Value OSINT Finds

| Finding | Impact | Tool |
|---------|--------|------|
| API keys in GitHub | Critical | truffleHog, gitrob |
| Staging subdomain | High | subfinder, amass |
| Old admin panel URL | High | Wayback Machine |
| Employee emails + breach data | High | theHarvester |
| Exposed Shodan service | Medium-High | Shodan |
| Tech stack version + CVE | High | whatweb |
| S3 bucket name in source | High | Google Dorks |

---

*Daily Bug Bounty notes — follow for more.*
