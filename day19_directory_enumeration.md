# Directory Enumeration — Gobuster, Dirsearch & Feroxbuster

> **Category:** Recon & Attack Surface Discovery  
> **Tools:** Gobuster, Dirsearch, Feroxbuster, ffuf  
> **Best For:** Finding hidden endpoints, admin panels, API routes, backup files  
> **Bounty Value:** Often leads to High/Critical — admin panels, exposed configs, sensitive files

---

## Why Directory Enumeration Matters

Web apps hide a LOT. Admin panels, dev endpoints, backup files, API routes — none of these appear in the sitemap. Directory brute forcing reveals the real attack surface. If you didn't scan for it, you missed it.

---

## Tool 1 — Gobuster (Fastest)

```bash
# Install
go install github.com/OJ/gobuster/v3@latest
apt install gobuster  # Kali

# Basic scan
gobuster dir -u https://target.com -w /opt/SecLists/Discovery/Web-Content/common.txt

# With file extensions
gobuster dir -u https://target.com -w big.txt -x php,html,js,txt,bak,old,zip

# Custom threads + output
gobuster dir -u https://target.com -w big.txt -t 50 -o gobuster_out.txt

# With auth cookie
gobuster dir -u https://target.com -w big.txt -c "session=abc123"

# HTTPS with custom headers
gobuster dir -u https://target.com -w big.txt -H "Authorization: Bearer TOKEN"

# DNS subdomain enumeration
gobuster dns -d target.com -w subdomains.txt

# S3 bucket enumeration
gobuster s3 -w bucket-names.txt
```

---

## Tool 2 — Dirsearch (Smartest)

```bash
# Install
pip install dirsearch --break-system-packages
git clone https://github.com/maurosoria/dirsearch

# Basic scan
dirsearch -u https://target.com

# With extensions
dirsearch -u target.com -e php,asp,aspx,jsp,html,js,json,xml

# Recursive scan (goes into found directories)
dirsearch -u target.com -r --max-recursion-depth 3

# Exclude status codes
dirsearch -u target.com --exclude-status 400,404,500

# Authenticated scan
dirsearch -u target.com -H "Cookie: session=abc123"

# Save output
dirsearch -u target.com -o report.txt --format plain
dirsearch -u target.com -o report.html --format html

# Scan multiple URLs
dirsearch -l urls.txt -e php,html
```

---

## Tool 3 — Feroxbuster (Deepest — Auto Recursive)

```bash
# Install
cargo install feroxbuster
apt install feroxbuster  # Kali

# Basic scan (auto-recursive!)
feroxbuster -u https://target.com -w wordlist.txt

# Filter noise
feroxbuster -u target.com --filter-status 404,403,400

# With extensions
feroxbuster -u target.com -x php,html,js -d 3

# Quiet mode (less output)
feroxbuster -u target.com --quiet

# Scan with cookies
feroxbuster -u target.com -H "Cookie: session=abc" -w wordlist.txt

# Limit recursion depth
feroxbuster -u target.com -d 2 -w wordlist.txt
```

---

## Best Wordlists (SecLists)

```bash
# Install SecLists
git clone https://github.com/danielmiessler/SecLists /opt/SecLists

# Quick scan (start here)
/opt/SecLists/Discovery/Web-Content/common.txt               # 4,713 entries

# Thorough scan
/opt/SecLists/Discovery/Web-Content/big.txt                  # 20,476 entries

# Best overall
/opt/SecLists/Discovery/Web-Content/raft-large-dirs.txt      # 62,284 entries
/opt/SecLists/Discovery/Web-Content/raft-large-files.txt     # 37,042 entries

# API specific
/opt/SecLists/Discovery/Web-Content/api/objects.txt
/opt/SecLists/Discovery/Web-Content/api/actions.txt

# Technology specific
/opt/SecLists/Discovery/Web-Content/PHP.fuzz.txt
/opt/SecLists/Discovery/Web-Content/Apache.fuzz.txt
/opt/SecLists/Discovery/Web-Content/IIS.fuzz.txt
```

---

## API Endpoint Discovery

```bash
# Gobuster on API base
gobuster dir -u https://target.com/api -w /opt/SecLists/Discovery/Web-Content/api/objects.txt

# Version enumeration
gobuster dir -u https://target.com -w - <<EOF
api
api/v1
api/v2
api/v3
internal
graphql
swagger
swagger-ui
EOF

# ffuf for API fuzzing (faster)
ffuf -u https://target.com/api/FUZZ -w /opt/SecLists/Discovery/Web-Content/api/objects.txt -mc 200,201,401,403

# Find API versions
ffuf -u https://target.com/FUZZ/users -w versions.txt -mc 200,401
```

---

## Filter & Fine-Tune Results

```bash
# Gobuster — exclude by response size (removes custom 404 pages)
gobuster dir -u target.com -w list.txt --exclude-length 1234

# feroxbuster — filter multiple status codes
feroxbuster -u target.com --filter-status 301,302,403,404

# ffuf — filter by size and status
ffuf -u target.com/FUZZ -w list.txt -fc 404 -fs 1234

# dirsearch — include only specific codes
dirsearch -u target.com -i 200,201,301,302,401,403

# Tip: Run once first to find the 404 page size, then filter it
```

---

## 403 Bypass Techniques

```bash
# When you find a 403, try these:
# Add headers
curl -H "X-Original-URL: /admin" target.com
curl -H "X-Rewrite-URL: /admin" target.com
curl -H "X-Custom-IP-Authorization: 127.0.0.1" target.com/admin

# Path manipulation
/admin          → 403
/admin/         → 200?  (trailing slash)
//admin/        → 200?  (double slash)
/ADMIN          → 200?  (uppercase)
/admin/.        → 200?  (dot)
/%2fadmin       → 200?  (URL encoded)
/admin%20       → 200?  (space encoded)

# Use tool: 403-bypass
python3 403bypass.py -u https://target.com/admin
```

---

## Full Recon Workflow

```bash
# Phase 1 — Quick scan (2 mins)
gobuster dir -u https://target.com -w common.txt -t 50 -o phase1.txt

# Phase 2 — Deep scan with extensions (10-20 mins)
gobuster dir -u https://target.com -w big.txt -x php,html,js,txt,bak,zip -t 30 -o phase2.txt

# Phase 3 — Recurse into interesting dirs found
feroxbuster -u https://target.com/admin -w raft-large-files.txt -d 3

# Phase 4 — API enumeration
gobuster dir -u https://target.com/api -w /opt/SecLists/Discovery/Web-Content/api/objects.txt

# Phase 5 — Review all 403s and try bypass techniques
```

---

## High Value Findings

```
/admin/             → Admin panel (try default creds)
/.git/              → Source code exposed
/.env               → Environment variables (passwords, keys)
/backup/            → Database dumps, source archives
/api/v1/admin       → Privileged API endpoint
/actuator/env       → Spring Boot secrets
/phpinfo.php        → PHP configuration
/server-status      → Apache server info
/swagger/           → Full API documentation
/graphql            → GraphQL endpoint (try introspection)
/config.php.bak     → Backup config files
/dump.sql           → Database dump
```

---

## Checklist

```
☐  Always start with common.txt then upgrade to big.txt
☐  Scan with extensions: -x php,html,js,txt,bak,old,zip
☐  Check every 403 — forbidden ≠ non-existent, try bypass!
☐  Recurse into found directories — /admin/ may have /admin/config/
☐  Use API wordlists on /api/, /v1/, /v2/ endpoints
☐  Filter by response size — same size responses = false positives
☐  Try feroxbuster for auto-recursion on promising targets
☐  Check robots.txt and sitemap.xml first for hints
```

---

## Report Template

```
Title: Exposed Admin Panel at /admin — No Authentication Required

Severity: Critical

Found via: gobuster dir -u https://target.com -w big.txt

Steps to Reproduce:
1. Run: gobuster dir -u https://target.com -w big.txt
2. Output: /admin [Status: 200] [Size: 4521]
3. Navigate to https://target.com/admin
4. Full admin dashboard accessible without authentication

Impact:
Unauthenticated access to admin panel — can create/delete users,
access all data, modify configuration, and fully compromise the app.

Remediation:
- Restrict /admin to authenticated users only
- Implement IP allowlisting for admin endpoints
- Add multi-factor authentication to admin access
```

---

## Tools Comparison

| Feature | Gobuster | Dirsearch | Feroxbuster | ffuf |
|---------|----------|-----------|-------------|------|
| Speed | ⚡⚡⚡ | ⚡⚡ | ⚡⚡⚡ | ⚡⚡⚡ |
| Recursive | ❌ | ✅ | ✅ Auto | Manual |
| Built-in wordlist | ❌ | ✅ | ❌ | ❌ |
| Language | Go | Python | Rust | Go |
| Best for | Quick recon | Smart scan | Deep enum | Fuzzing |

---

*Daily Bug Bounty notes — follow for more.*
