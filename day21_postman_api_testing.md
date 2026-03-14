# Postman — API Testing for Bug Hunters

> **Category:** API Security Testing  
> **Tool:** Postman (Free)  
> **Best For:** IDOR, Auth Bypass, Mass Assignment, Rate Limit Testing  
> **Bounty Value:** High/Critical — API bugs are the most rewarding in modern programs

---

## Why Postman for Bug Bounty?

APIs are the new attack surface. Every mobile app, SPA, and microservice communicates via APIs — and most are undertested. Postman lets you intercept, modify, chain, and automate API requests with zero setup. It's the fastest way to find IDOR, BOLA, auth bypass, and business logic flaws.

---

## 01 · Setting Up Collections

```
1. New Collection → name it after the target
2. Add Requests → set method (GET/POST/PUT/DELETE/PATCH)
3. Set Environment Variables:
   - {{base_url}}   = https://api.target.com
   - {{token}}      = Bearer eyJhbGc...  (your auth token)
   - {{user_id}}    = 1002              (your account ID)
   - {{other_id}}   = 1001              (victim account ID)

4. Pre-request Script (auto-set auth):
   pm.request.headers.add({
     key: 'Authorization',
     value: 'Bearer ' + pm.environment.get('token')
   });
```

**Pro tip:** Use two environments — Account A and Account B — to test cross-account access (IDOR/BOLA) instantly.

---

## 02 · Authentication Testing

```
Test 1 — No auth at all
  DELETE /api/v1/user/{{user_id}}
  → Remove Authorization header entirely
  → Expected: 401 Unauthorized
  → Bug if: 200 OK or 204 No Content

Test 2 — Another user's token
  GET /api/v1/account/{{other_id}}/details
  → Use Account A token to access Account B's endpoint
  → Expected: 403 Forbidden
  → Bug if: 200 OK returns another user's data

Test 3 — Expired / malformed JWT
  Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VySWQiOiIxIn0.
  → Set alg:none in JWT header
  → Expected: 401
  → Bug if: API accepts unsigned JWT

Test 4 — JWT role manipulation
  Decode JWT → change "role":"user" to "role":"admin" → re-sign or use none algo
  → Expected: 403
  → Bug if: server trusts client-side role claim
```

---

## 03 · IDOR & BOLA Testing

```
# The Golden IDOR Checklist:

# 1. Path parameter ID change
GET /api/v1/invoices/1001      ← your invoice
GET /api/v1/invoices/1000      ← try decrement
GET /api/v1/invoices/1002      ← try increment

# 2. Request body ID swap
PUT /api/v1/profile
Body: {"user_id": 9999, "email": "attacker@evil.com"}

# 3. Nested resource IDOR
GET /api/v1/users/1001/orders      ← should be 403 for user 1002

# 4. GUID/UUID still vulnerable
GET /api/v1/documents/550e8400-e29b-41d4-a716-446655440000
→ Find another UUID via email, error message, or enumeration

# 5. HTTP parameter pollution
GET /api/v1/user?id=1002&id=1001   ← which one does the API use?

# Automation in Postman:
# Tests tab → check for another user's data in response
pm.test("No IDOR", function() {
  var data = pm.response.json();
  pm.expect(data.email).to.not.equal("victim@target.com");
});
```

---

## 04 · Mass Assignment

```
# Find object properties in GET response:
GET /api/v1/users/me
Response: {"id":1002,"email":"me@me.com","role":"user","is_verified":true}

# Now try sending those fields in POST/PUT:
POST /api/v1/register
Body: {
  "email": "attacker@evil.com",
  "password": "Test1234!",
  "role": "admin",           ← add this
  "is_verified": true,       ← add this
  "credits": 99999           ← add this
}

# Common mass assignment targets:
- role / user_role / account_type
- is_admin / admin / superuser
- is_verified / email_verified
- balance / credits / wallet_amount
- subscription / plan / tier

# In Postman: use Body tab → raw → JSON
# Add every field from GET response back into POST/PUT
```

---

## 05 · Rate Limit & Business Logic

```
# Test rate limiting with Collection Runner:
1. Create request: POST /api/v1/login
   Body: {"email":"victim@target.com","password":"{{password}}"}
2. Collection Runner → select collection
3. Iterations: 100 | Delay: 0ms
4. Data file: passwords.csv
→ If no 429 returned after 10+ attempts = rate limit bypass

# Race condition (Turbo Intruder approach via Postman):
1. Add 50 copies of same request (e.g., redeem coupon)
2. Collection Runner → run all simultaneously
→ Check if coupon applied multiple times

# Business logic test examples:
- Apply discount → cancel order → does discount code reactivate?
- Buy item for $0.00 → intercept → change price to -1
- Refer yourself → do referral credits stack?
- Upload file → tamper filename → path traversal?
```

---

## 06 · Chaining Requests & Automation

```javascript
// Tests tab — auto-save token after login:
var data = pm.response.json();
pm.environment.set("token", data.access_token);
pm.environment.set("user_id", data.user.id);
pm.environment.set("refresh_token", data.refresh_token);

// Pre-request script — auto-refresh expired token:
var token = pm.environment.get("token");
if (!token) {
  pm.sendRequest({
    url: pm.environment.get("base_url") + "/auth/refresh",
    method: "POST",
    body: { refresh_token: pm.environment.get("refresh_token") }
  }, function(err, res) {
    pm.environment.set("token", res.json().access_token);
  });
}

// Test assertion — detect data leaks:
pm.test("No other user's data", function() {
  var body = pm.response.text();
  pm.expect(body).to.not.include("victim@target.com");
  pm.expect(body).to.not.include("admin@target.com");
});

// Collection Runner attack chain:
// 1. Register (Account A)  → save token_a
// 2. Register (Account B)  → save token_b
// 3. Create resource (A)   → save resource_id
// 4. Access resource (B)   → should fail → if 200 = IDOR!
```

---

## Full API Bug Hunting Workflow

```
Step 1 — Discover endpoints
  • Crawl app with Burp Suite → export to Postman
  • Import Swagger/OpenAPI spec if available
  • Check: /swagger, /api-docs, /openapi.json, /v1/docs

Step 2 — Set up environments
  • Create Account A (normal user)
  • Create Account B (normal user)
  • Note both user IDs and tokens

Step 3 — Test auth on every endpoint
  • Remove Authorization header → expect 401
  • Use Account B token on Account A's resources → expect 403

Step 4 — Test IDOR on every ID field
  • Path params, body params, query params
  • Increment, decrement, use Account B's IDs

Step 5 — Mass assignment on all POST/PUT
  • Add role, is_admin, user_id, balance to every body

Step 6 — Rate limit on sensitive endpoints
  • /login, /forgot-password, /verify-otp, /apply-coupon

Step 7 — Document and report
  • Screenshot Postman request + response
  • Note the exact header/body change that caused the bug
```

---

## Checklist

```
☐  Remove auth header — every endpoint should return 401/403
☐  Change user IDs — increment/decrement in every request
☐  Add extra params — role, is_admin, user_id in every POST
☐  Test rate limits — send 50-100 requests, check for 429
☐  Try other users' JWTs — Account B token on Account A endpoints
☐  Fuzz HTTP methods — GET/POST/PUT/DELETE/PATCH on every route
☐  Check Swagger/OpenAPI — hidden endpoints in API docs
☐  Test expired tokens — should always return 401
```

---

## High Value API Findings

| Bug | How to Find | Severity |
|-----|-------------|----------|
| IDOR | Change ID in path/body | High/Critical |
| BOLA | Use another user's token | Critical |
| Mass Assignment | Add role/admin to POST body | High |
| Auth Bypass | Remove Authorization header | Critical |
| Rate Limit Missing | 100 requests, no 429 | Medium/High |
| JWT None Algorithm | Set alg:none | Critical |
| Broken Object Property Level | Guess hidden properties | Medium |

---

*Daily Bug Bounty notes — follow for more.*
