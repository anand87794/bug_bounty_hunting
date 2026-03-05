# Security Misconfiguration

> **OWASP:** A05:2021  
> **Severity:** Low → Critical (depends on what's exposed)  
> **Bounty Range:** $100 – $25,000+ (secrets/cloud = Critical)  
> **Why it matters:** #1 most commonly found issue in real-world pentests

---

## What is Security Misconfiguration?

When security settings are defined, implemented, or maintained incorrectly — or not at all. This includes default configs, unnecessary features enabled, missing hardening, and verbose error messages. It's the easiest class of bugs to find and often leads directly to critical impact.

---

## 6 Attack Vectors

### 01 · Default Credentials
```
Try these on every login page you find:
admin:admin
admin:password
admin:1234
root:root
root:toor
admin:admin123
guest:guest
test:test

Common targets:
/admin          /wp-admin       /phpmyadmin
/manager        /console        /dashboard
/cpanel         /webadmin       /administrator

Tool: Use Burp Intruder with default-credentials wordlist
```

### 02 · Directory Listing Enabled
```
Add / to paths to trigger directory listing:
https://target.com/uploads/
https://target.com/backup/
https://target.com/files/
https://target.com/static/
https://target.com/assets/

What to look for:
- .env files (DB passwords, API keys)
- database.sql / db_backup.sql
- config.php / settings.py / application.properties
- old .zip / .tar.gz archives of source code
```

### 03 · Verbose Error Messages
```
How to trigger errors:
- Send malformed JSON:    {"key": }
- SQL injection chars:    ' OR 1=1--
- Oversized payloads:     A * 10000
- Wrong content-type:     Send XML to JSON endpoint
- Missing required fields: Remove required params

What errors reveal:
- Framework + version: "Django 2.1.3", "Laravel 8.0"
- DB type + query:     "MySQL syntax error near..."
- Absolute file paths: "/var/www/app/models/user.php line 42"
- Internal IP ranges:  "Connection refused: 10.0.0.5:5432"
- Stack traces:        Full call stack = architecture map
```

### 04 · Exposed Debug Endpoints
```bash
# Spring Boot Actuator (HIGH VALUE — leaks everything)
/actuator               # Lists all available endpoints
/actuator/env           # ALL environment variables (passwords, keys!)
/actuator/mappings      # All URL routes in the app
/actuator/beans         # All Spring beans
/actuator/heapdump      # Full memory dump — extract secrets offline

# General debug endpoints
/.git/                  # Source code exposure
/.git/config            # Remote URL + credentials
/.env                   # All environment variables
/phpinfo.php            # PHP config, extensions, paths
/server-status          # Apache server status
/api/swagger            # Full API documentation
/api/swagger-ui.html    # Interactive API explorer
/graphql                # GraphQL endpoint (try introspection)
/console                # Rails/Django admin console

# Test for GraphQL introspection
POST /graphql
{"query": "{ __schema { types { name } } }"}
```

### 05 · Missing Security Headers
```bash
# Check headers with curl
curl -I https://target.com

# Headers to look for (missing = finding):
Content-Security-Policy      # Missing = XSS easier to exploit
X-Frame-Options              # Missing = clickjacking possible
X-Content-Type-Options       # Missing = MIME sniffing attacks
Strict-Transport-Security    # Missing = SSL stripping possible
Referrer-Policy              # Missing = URL leakage in requests
Permissions-Policy           # Missing = camera/mic/geo access

# Report template for missing headers:
# Low severity alone, but can chain with other bugs
# Example: Missing CSP + XSS = higher severity report
```

### 06 · Cloud Storage Misconfiguration
```bash
# S3 Bucket enumeration
https://target.s3.amazonaws.com/
https://s3.amazonaws.com/target/
https://target-backup.s3.amazonaws.com/

# AWS CLI test
aws s3 ls s3://target-bucket --no-sign-request
aws s3 cp s3://target-bucket/secret.txt . --no-sign-request

# Azure Blob
https://target.blob.core.windows.net/container/?restype=container&comp=list

# GCP Storage
https://storage.googleapis.com/target-bucket/

# Where to find bucket names:
# - JS source files (grep for s3.amazonaws.com)
# - HTML source comments
# - API response fields
# - Mobile app strings
# - Job listings (tech stack mentions)
```

---

## Quick Recon Checklist

```bash
# 1. Check robots.txt and sitemap
curl https://target.com/robots.txt
curl https://target.com/sitemap.xml

# 2. Common sensitive files
curl -s -o /dev/null -w "%{http_code}" https://target.com/.env
curl -s -o /dev/null -w "%{http_code}" https://target.com/.git/config
curl -s -o /dev/null -w "%{http_code}" https://target.com/backup.zip

# 3. Response headers check
curl -I https://target.com | grep -iE "server|x-powered-by|x-frame|csp|hsts"

# 4. Directory brute force
gobuster dir -u https://target.com -w /usr/share/seclists/Discovery/Web-Content/common.txt

# 5. S3 bucket finder
python3 s3scanner.py --bucket target
```

---

## Checklist

```
☐  Try default creds on every login page: admin/admin, root/root
☐  Test /.env  /.git/  /backup/  /actuator  /swagger  /phpinfo.php
☐  Check HTTP security headers: curl -I target.com
☐  Trigger errors with invalid input — look for stack traces
☐  Search JS source for S3 bucket names or cloud storage URLs
☐  Test directory listing: add / to every path you find
☐  Try /actuator/env on Spring Boot apps — leaks all secrets
☐  Check robots.txt and sitemap.xml for hidden paths
```

---

## Impact Examples

| Misconfiguration | Impact | Severity |
|-----------------|--------|----------|
| admin:admin works | Full admin access | Critical |
| .env file exposed | DB password, API keys | Critical |
| /actuator/env open | All secrets in memory | Critical |
| Public S3 bucket | All stored files accessible | High |
| Directory listing | Config files, backups found | Medium |
| Verbose errors | Tech stack + path disclosure | Low-Medium |
| Missing CSP | XSS easier to exploit | Low (chains to High) |

---

## Report Template

```
Title: Exposed .env File Contains Database Credentials

Severity: Critical (CVSS 9.8)

URL: https://target.com/.env

Steps to Reproduce:
1. Navigate to https://target.com/.env
2. File returns 200 OK with contents:
   DB_PASSWORD=SuperSecret123
   AWS_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
   STRIPE_SECRET=sk_live_abc123...

Impact:
Full database access, AWS account compromise,
payment processor access — complete infrastructure takeover.

Remediation:
- Block access to .env files via web server config
- Rotate all exposed credentials immediately
- Add .env to .gitignore and web server deny rules
```

---

## Tools

- **Nikto** — `nikto -h https://target.com` — scans for misconfigs automatically
- **Gobuster** — directory/file brute force
- **S3Scanner** — find and test S3 buckets
- **SecurityHeaders.com** — check response headers
- **Shodan** — find exposed panels with default creds

---

*Daily Bug Bounty notes — follow for more.*
