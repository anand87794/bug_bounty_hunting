# HTTP Security Headers

> **OWASP:** A05:2021 — Security Misconfiguration  
> **Severity:** Low → High (depends on what's missing + what chains)  
> **Quick check:** `curl -I https://target.com` or securityheaders.com

---

## What are Security Headers?

HTTP response headers that tell the browser how to behave securely. Missing headers = browser has no restrictions = easy attack surface. One `curl -I` command can reveal multiple findings on any target.

---

## 6 Critical Headers

### 01 · Content-Security-Policy (CSP)
```
What it does: Controls which scripts, styles, fonts, images can load.
Missing = XSS payloads execute without any restriction.

Secure example:
Content-Security-Policy: default-src 'self'; script-src 'self'; 
  object-src 'none'; base-uri 'self';

Bug hunting tips:
- Missing CSP entirely = report + test for XSS immediately
- unsafe-inline in script-src = CSP bypass possible
- unsafe-eval = DOM XSS easier to exploit
- wildcard (*) sources = too permissive

Test CSP bypass:
https://csp-evaluator.withgoogle.com/
```

### 02 · Strict-Transport-Security (HSTS)
```
What it does: Forces HTTPS — browser never uses HTTP again.
Missing = SSL stripping attack possible (HTTP MitM).

Secure example:
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Key values:
max-age=31536000   = 1 year (minimum recommended)
includeSubDomains  = applies to all subdomains too
preload            = submit to browser preload list (hardcoded HTTPS)

Missing HSTS attack:
1. Victim visits http://target.com (not https)
2. Attacker intercepts and strips SSL
3. Victim browses in HTTP — credentials visible in plaintext
```

### 03 · X-Frame-Options
```
What it does: Prevents site from being loaded inside an iframe.
Missing = clickjacking attack possible.

Values:
X-Frame-Options: DENY          # No framing at all (most secure)
X-Frame-Options: SAMEORIGIN    # Only same domain can frame

Clickjacking PoC (when missing):
<html>
<body>
  <iframe src="https://target.com/transfer" style="opacity:0; position:absolute; top:0; left:0; width:100%; height:100%;"></iframe>
  <button style="position:absolute; top:200px; left:300px;">Win a Prize!</button>
</body>
</html>

Note: CSP frame-ancestors replaces X-Frame-Options in modern browsers
CSP: frame-ancestors 'none'  =  equivalent to DENY
```

### 04 · X-Content-Type-Options
```
What it does: Stops browser from MIME-sniffing content type.
Missing = upload a .jpg containing JS → browser executes it as script.

Only one value:
X-Content-Type-Options: nosniff

Attack without it:
1. Upload file.jpg containing: <script>alert(1)</script>
2. Browser sniffs content → sees JS → executes it!
3. With nosniff: browser respects Content-Type: image/jpeg → safe

Easiest fix: one line, zero performance impact.
```

### 05 · Referrer-Policy
```
What it does: Controls how much URL info is sent in Referer header.
Missing = sensitive URLs leak to third-party analytics/CDN.

Values (most → least restrictive):
no-referrer                    # Never send Referer
same-origin                    # Only for same-origin requests
strict-origin                  # Only send domain (no path)
strict-origin-when-cross-origin # Domain only for cross-origin (recommended)
no-referrer-when-downgrade     # Default browser behavior

Attack scenario without it:
User visits: https://target.com/reset-password?token=SECRET123
Page loads Google Analytics → Referer header sent:
Referer: https://target.com/reset-password?token=SECRET123
→ Token leaked to Google!
```

### 06 · Permissions-Policy
```
What it does: Restrict browser feature access per page.
Missing = if XSS hits the page, attacker can access camera/mic/geo.

Secure example:
Permissions-Policy: camera=(), microphone=(), geolocation=(), 
  payment=(), usb=(), magnetometer=()

() = disable entirely for all origins
self = allow only same origin
* = allow all (insecure)

Replaces old Feature-Policy header.
```

---

## Quick Recon Commands

```bash
# Check all headers
curl -I https://target.com

# Filter for security headers only
curl -sI https://target.com | grep -iE "content-security|strict-transport|x-frame|x-content-type|referrer|permissions|feature"

# Online scanner (gives A-F grade)
# Visit: https://securityheaders.com/?q=target.com

# Check with nikto
nikto -h https://target.com | grep -i header
```

---

## Checklist

```
☐  curl -I https://target.com — list all response headers
☐  Check securityheaders.com — instant A-F grade report
☐  Missing CSP? Test XSS — inline scripts likely execute freely
☐  Missing HSTS? Try SSL strip attack on HTTP version
☐  Missing X-Frame-Options? Build clickjacking PoC iframe
☐  Missing X-Content-Type-Options? Test MIME sniff with file upload
☐  Report each missing critical header as separate Low/Medium finding
☐  Check if CSP has unsafe-inline or * wildcard = bypass possible
```

---

## Severity Guide

| Header | Missing Impact | Severity |
|--------|---------------|----------|
| CSP | XSS unrestricted | Medium (chains to High) |
| HSTS | SSL strip / MitM | Medium |
| X-Frame-Options | Clickjacking | Low-Medium |
| X-Content-Type-Options | MIME sniff attack | Low |
| Referrer-Policy | URL/token leakage | Low-Medium |
| Permissions-Policy | Feature abuse via XSS | Low |

---

## Report Template

```
Title: Missing Content-Security-Policy Header

Severity: Medium (Low standalone, chains to High with XSS)

URL: https://target.com

Steps to Reproduce:
1. curl -I https://target.com
2. Observe: no Content-Security-Policy header in response

Impact:
Without CSP, any successful XSS attack can execute arbitrary
JavaScript with no browser-level restrictions. Attackers can
steal cookies, redirect users, and perform actions on behalf
of the victim.

Remediation:
Add: Content-Security-Policy: default-src 'self'; script-src 'self';
object-src 'none'; base-uri 'self';
```

---

## Tools

- **securityheaders.com** — instant visual grade report
- **CSP Evaluator** — csp-evaluator.withgoogle.com
- **Mozilla Observatory** — observatory.mozilla.org
- **curl** — `curl -I target.com`

---

*Daily Bug Bounty notes — follow for more.*
