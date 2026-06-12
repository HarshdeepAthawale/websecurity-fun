# A05 — Security Misconfiguration

← [[OWASP-Overview]]

---

Security misconfiguration is the most commonly found issue in practice. It covers every case where a system is set up in an insecure way — default settings left unchanged, unnecessary features enabled, verbose error messages, cloud misconfigs, and more.

---

## Common Misconfiguration Types

### Default Credentials

Admin interfaces, routers, databases, and management tools ship with default username/password combinations.

```
admin:admin
admin:password
admin:123456
root:root
root:(blank)
```

**Test:** Any newly discovered admin panel or management interface — try defaults. Reference: [Default Credentials Cheat Sheet](https://github.com/ihebski/DefaultCreds-cheat-sheet)

---

### Verbose Error Messages

Stack traces and detailed error messages leak:
- Technology stack (PHP, Java, Python)
- Framework and version
- File paths on the server
- Database queries
- Internal IP addresses

```bash
# Trigger errors with invalid input
curl "https://example.com/api/user/INVALID"
curl "https://example.com/api/user?id='"
```

**Finding:** If you see a full stack trace or raw SQL error in the response — that's a misconfiguration finding (and a hint for further attacks).

---

### Directory Listings

Web server configured to show file listings when no `index.html` exists.

```
Index of /uploads/
Name                    Modified           Size
../
profile_123.jpg         2023-01-15         45K
report_admin_2023.pdf   2023-03-20         2M
backup.zip              2022-12-01         150M
```

**Test:** Try subdirectories you find during recon. A directory listing on `/uploads/` or `/backup/` is a critical finding.

---

### Exposed Admin Interfaces

```
/admin
/administrator
/phpmyadmin
/phpMyAdmin
/manager/html          ← Tomcat Manager
/_admin
/wp-admin              ← WordPress
/controlpanel
/cpanel
```

If accessible without auth or with defaults = critical. If accessible to the internet at all = misconfiguration.

---

### Unnecessary Features Enabled

- **phpinfo()** page — leaks PHP configuration, environment variables, loaded modules
- **Debug mode** in production — `DEBUG=True` in Django, `NODE_ENV=development` in Node
- **Swagger/API documentation** exposed publicly with sensitive endpoint details
- **GraphQL introspection** enabled in production — reveals the full schema
- **Server status pages** — `/server-status` (Apache), `/nginx_status`
- **Git repository** exposed — `/.git/` accessible

```bash
# Quick checks
curl https://example.com/phpinfo.php
curl https://example.com/.git/config
curl https://example.com/server-status
curl https://example.com/api-docs
curl https://example.com/swagger.json
```

---

### Cloud Misconfiguration

The biggest source of data breaches in recent years.

**S3 Buckets:**
```bash
# Check if a bucket is publicly readable
aws s3 ls s3://company-bucket --no-sign-request

# Try the bucket name variations
https://company.s3.amazonaws.com/
https://s3.amazonaws.com/company/
https://company-backup.s3.amazonaws.com/
```

**Exposed services (from Shodan recon):**
- Elasticsearch on port 9200 with no auth — database fully readable
- Redis on port 6379 with no auth — can read/write all data, sometimes RCE
- MongoDB on port 27017 with no auth
- Kibana on port 5601 with no auth

---

### Missing Security Headers

See [[HTTP-Headers]] for the full list. At minimum:
- `Content-Security-Policy`
- `Strict-Transport-Security`
- `X-Frame-Options` or `frame-ancestors`
- `X-Content-Type-Options: nosniff`

Missing headers are usually low/informational by themselves but critical in combination with other vulns.

---

### CORS Misconfiguration

See [[CORS]].

---

### Outdated / Default SSL/TLS Configuration

- Supporting TLS 1.0 / 1.1 (deprecated)
- Weak cipher suites (RC4, 3DES, EXPORT ciphers)
- Self-signed or expired certificates

```bash
testssl.sh https://example.com
nmap --script ssl-enum-ciphers -p 443 example.com
```

---

## Testing Checklist

- [ ] Default credentials on any admin panel?
- [ ] Verbose error messages triggered by invalid input?
- [ ] Directory listings on any path?
- [ ] `/.git/`, `/.env`, `/phpinfo.php`, `/server-status` accessible?
- [ ] Debug mode enabled in production?
- [ ] Swagger / API docs exposed?
- [ ] GraphQL introspection enabled?
- [ ] S3 buckets publicly readable?
- [ ] Exposed databases on Shodan?
- [ ] Security headers present?
- [ ] TLS version and cipher strength?

---

*Related: [[Fingerprinting]] | [[Directory-Bruteforce]] | [[OSINT]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, OWASP Testing Guide v4*
