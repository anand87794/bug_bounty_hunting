# Burp Suite Workflow

> **Category:** Tools & Techniques — Week 3  
> **Tool:** Burp Suite Pro / Community  
> **Download:** https://portswigger.net/burp

---

## Setup in 3 Steps

```
1. Set browser proxy → 127.0.0.1:8080
2. Visit http://burp → download CA certificate
3. Install CA cert in browser → HTTPS traffic now visible
```

Firefox (easiest): Settings → Network → Manual proxy → 127.0.0.1:8080  
Or use FoxyProxy extension for quick toggle.

---

## 6 Core Tools

### 01 · Proxy — Intercept Everything
```
Sits between browser and server — see and modify every request/response.

Key controls:
- Intercept ON  → every request pauses for you to inspect/modify
- Intercept OFF → traffic flows normally (passive logging)
- HTTP history  → full log of all requests made

Workflow:
1. Turn Intercept ON
2. Browse to login, profile, payment page
3. Inspect every parameter — what's user-controlled?
4. Modify anything suspicious → Forward
```

### 02 · Repeater — Manual Fuzzing
```
Resend any request with your own modifications. Core manual testing tool.

How to use:
1. Find interesting request in Proxy → HTTP history
2. Right-click → Send to Repeater (Ctrl+R)
3. Modify: change user ID, inject payload, swap cookies
4. Click Send → inspect response
5. Repeat with variations

Pro tips:
- Compare response length changes — shorter/longer = something changed
- Use multiple Repeater tabs for different endpoints
```

### 03 · Intruder — Automated Fuzzing
```
Blasts a wordlist at marked positions in a request.

Attack types:
Sniper       → one payload position, iterates list one by one
Battering Ram → same payload in ALL positions simultaneously
Pitchfork    → multiple lists, one per position (paired 1:1)
Cluster Bomb → all combinations of all lists (very loud)

Workflow:
1. Send request to Intruder (Ctrl+I)
2. Positions tab → clear § → highlight target param → Add §
3. Payloads tab → load wordlist
4. Start Attack → sort results by Status / Length
5. Anomaly in response = potential vulnerability

Note: Community edition rate-limits Intruder. Use Turbo Intruder
extension for full speed, or ffuf/feroxbuster externally.
```

### 04 · Scanner — Passive + Active Scan
```
Auto-detects 100+ vulnerability types.

Passive: runs in background automatically — no extra requests
Active:  sends attack payloads — finds real vulns (Pro only)

How to use:
1. Right-click any request → Scan → Active Scan
2. Dashboard → Issues tab → sorted by severity
3. Click issue → PoC request + remediation advice

Finds: SQLi, XSS, SSRF, path traversal, XXE, open redirects,
       missing headers, weak SSL, clickjacking, and 100+ more
```

### 05 · Target / Site Map — Enumerate Attack Surface
```
Builds full visual map of all discovered content.

Workflow:
1. Browse entire app with Intercept OFF
2. Site Map fills with every endpoint visited
3. Right-click domain → Spider for deeper crawl

Filter for:
- /admin /api /debug /backup /test /internal endpoints
- .json .xml .config .bak .old file extensions
- Items with parameters (your attack surface)
- Compare auth vs unauth map → broken access control?
```

### 06 · Decoder + Comparer
```
Decoder:
- Paste JWT/cookie → base64 decode → read hidden claims
- Chain decoders: URL decode → base64 → JSON
- Encode payloads for WAF bypass
- Ctrl+Shift+D = send selection to Decoder

Comparer:
- Paste authenticated response vs unauthenticated response
- Words view: spot semantic differences
- Bytes view: exact byte diff
- Use case: IDOR — does removing auth cookie change response?
```

---

## Must-Have Extensions (BApp Store)

| Extension | What it does |
|-----------|-------------|
| Turbo Intruder | Full-speed fuzzing — bypasses Community rate limit |
| Autorize | Auto-test access control as lower-privilege user |
| JWT Editor | alg:none, key confusion, all JWT attacks |
| Logger++ | Advanced request filtering and logging |
| Param Miner | Discover hidden / unlinked parameters |
| Active Scan++ | Extra active scan checks |

---

## Keyboard Shortcuts

```
Ctrl+R  = Send to Repeater
Ctrl+I  = Send to Intruder
Ctrl+D  = Send to Decoder
Ctrl+M  = Send to Comparer
Ctrl+F  = Forward intercepted request
```

---

## Workflow Checklist

```
☐  Set proxy 127.0.0.1:8080 + install CA cert in browser
☐  Browse whole target with Intercept OFF — build Site Map
☐  Interesting request? Right-click → Send to Repeater
☐  Repeater finds something? Send to Intruder → fuzz it
☐  Run Active Scan on login, search, upload endpoints
☐  Weird token or cookie? Decoder → base64 decode → read it
☐  Check Site Map for /admin /api /debug /backup endpoints
☐  Install Autorize → test every endpoint as low-priv user
```

---

*Daily Bug Bounty notes — follow for more.*
