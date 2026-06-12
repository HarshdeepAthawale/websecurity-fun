# A02 — Cryptographic Failures

← [[OWASP-Overview]]

---

Cryptographic failures happen when sensitive data is not adequately protected — either it's transmitted or stored without encryption, or the encryption used is weak/broken.

---

## What Falls Under This Category

- Sensitive data transmitted in plaintext (HTTP instead of HTTPS)
- Sensitive data stored without encryption (passwords in plaintext, PII in plain DB columns)
- Weak or outdated cryptographic algorithms (MD5, SHA1, DES, RC4)
- Weak keys or poor key management
- Missing certificate validation (ignoring TLS errors)
- Hardcoded cryptographic keys or secrets

---

## Sensitive Data in Transit

The most basic check: is the app using HTTPS?

```bash
# Does the app redirect HTTP to HTTPS?
curl -sI http://example.com | grep -i location

# Is HSTS set?
curl -sI https://example.com | grep -i "strict-transport"
```

**Also check:**
- Are cookies sent over HTTP? (Missing `Secure` flag)
- Does the app load any resources (images, scripts) over HTTP on an HTTPS page? (Mixed content)
- Can you downgrade the TLS version? (`SSL 3.0`, `TLS 1.0`, `TLS 1.1` are deprecated)

```bash
# Check supported TLS versions
nmap --script ssl-enum-ciphers -p 443 example.com
testssl.sh https://example.com
```

---

## Sensitive Data in Storage

### Plaintext Passwords

The classic critical finding. If an app stores passwords in plaintext in its database, any database breach = all accounts compromised.

**How to test:** Register with password `hunter2`. If the app can "send you your password" via email (not a reset link, the actual password), it's stored plaintext or reversibly encrypted.

### Weak Password Hashing

Passwords should be hashed with a **slow, salted** algorithm: bcrypt, scrypt, Argon2, or PBKDF2.

**Weak algorithms in use:**
- `MD5($password)` — cracks in seconds with GPU
- `SHA1($password)` — fast hash, not designed for passwords
- `MD5($salt.$password)` — better but still crackable
- Unsalted hashes — rainbow table attacks work

**How to identify from a breach/SQLi dump:**

| Format | Algorithm |
|--------|-----------|
| `5f4dcc3b5aa765d61d8327deb882cf99` (32 hex chars) | MD5 |
| `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` (40 hex) | SHA1 |
| `$2y$10$...` | bcrypt ✅ |
| `$argon2id$...` | Argon2id ✅ |
| Plain text visible | Plaintext ❌ |

---

## Weak or Broken Algorithms

| Algorithm | Status | Issue |
|-----------|--------|-------|
| MD5 | Broken | Collision attacks, fast cracking |
| SHA1 | Deprecated | Collision demonstrated (SHAttered) |
| DES / 3DES | Broken | Key size too small |
| RC4 | Broken | Biases in keystream |
| ECB mode (any cipher) | Broken | Identical plaintext → identical ciphertext, no IV |
| AES-CBC without auth | Weak | Padding oracle attacks |
| RSA < 2048 bits | Weak | Factorizable |

---

## Hardcoded Keys and Secrets

Cryptographic keys hardcoded in source code are a critical finding — especially if the code is open-source or the repo was ever public.

```bash
# Find hardcoded keys in JS files
grep -rE "(secret|private_key|api_key|encryption_key|jwt_secret)\s*[=:]\s*['\"][^'\"]{8,}" .

# AWS keys
grep -rE "AKIA[0-9A-Z]{16}" .

# Private key headers
grep -rE "-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----" .
```

---

## Testing for Cryptographic Failures

| Check | How |
|-------|-----|
| HTTP→HTTPS redirect | `curl -sI http://example.com` |
| HSTS header | `curl -sI https://example.com \| grep strict` |
| Secure cookie flag | Check `Set-Cookie` headers |
| TLS version / cipher strength | `testssl.sh`, `nmap --script ssl-enum-ciphers` |
| Password storage | Register + check "forgot password" flow |
| Sensitive data in URLs | Check if tokens/passwords appear in request URLs (visible in logs/referer) |
| Source map exposure | `https://example.com/static/js/main.js.map` |
| Backup files | `/backup.sql`, `/dump.sql` in bruteforce |

---

## Common Real-World Findings

- Password reset tokens sent in URL (visible in logs and `Referer` header)
- JWT signed with a weak secret crackable with `hashcat`
- API keys in JavaScript source code or git history
- Database backups accessible without authentication
- Credit card numbers stored in plaintext in order database
- Internal API communicating over HTTP between microservices

---

*Related: [[JWT-Attacks]] | [[HTTP-Cookies-and-Sessions]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, OWASP Cryptographic Storage Cheat Sheet*
