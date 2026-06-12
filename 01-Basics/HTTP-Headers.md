# HTTP Headers Reference

← [[HTTP-Deep-Dive]]

---

Headers are key-value pairs in HTTP requests and responses that carry metadata. From a security perspective, headers are both **attack vectors** (headers you send to exploit) and **defenses** (headers the server sets to protect users).

---

## Request Headers — Attacker's Toolkit

These are headers you control when crafting or modifying a request.

### Identity & Routing

| Header | Purpose | Security Abuse |
|--------|---------|---------------|
| `Host` | Which virtual host to serve | **Host header injection** — password reset links, cache poisoning. Try changing it to an attacker domain. |
| `X-Forwarded-For` | Real IP behind a proxy | **IP spoofing** — bypass IP-based rate limiting or allowlists with `X-Forwarded-For: 127.0.0.1` |
| `X-Real-IP` | Real IP (nginx convention) | Same as above |
| `X-Forwarded-Host` | Original host behind proxy | Override routing logic, poison caches |
| `X-Original-URL` | Override the URL processed internally | **403 bypass** — `GET /sensitive` blocked? Try `GET / HTTP/1.1` + `X-Original-URL: /sensitive` |
| `X-Rewrite-URL` | Same as above (IIS) | 403 bypass |

> [!TIP]
> Host header injection is most powerful in password reset flows. If the app builds the reset link using the `Host` header without validation, you can make it send `https://attacker.com/reset?token=...` to the victim.

### Authentication

| Header | Purpose | Security Abuse |
|--------|---------|---------------|
| `Authorization` | Credentials (Bearer token, Basic auth) | Token theft, JWT attacks, weak secrets |
| `Cookie` | Session cookies | Session hijacking |
| `X-API-Key` | API key auth | Key leakage in logs, JS files, git history |

### Content

| Header | Purpose | Security Abuse |
|--------|---------|---------------|
| `Content-Type` | Format of request body | Change `application/json` to `text/xml` → XXE. Change to `application/x-www-form-urlencoded` → bypass JSON-only WAF rules |
| `Content-Length` | Size of body | HTTP request smuggling via CL desync |
| `Transfer-Encoding` | How body is encoded (chunked) | HTTP request smuggling via TE desync |
| `Accept` | What response format the client wants | Content negotiation abuse |

### Cross-Origin

| Header | Purpose | Security Abuse |
|--------|---------|---------------|
| `Origin` | Source origin for CORS | Set to attacker origin to test CORS misconfig |
| `Referer` | Previous page URL | Can be used for CSRF bypass (if app checks it), also leaks sensitive URLs |

### Miscellaneous

| Header | Purpose | Security Abuse |
|--------|---------|---------------|
| `User-Agent` | Browser/client identifier | WAF fingerprinting/bypass (some WAFs are lenient with Googlebot UA) |
| `X-Requested-With: XMLHttpRequest` | Marks AJAX requests | Some apps skip CSRF checks if this header is present |

---

## Response Headers — Security Defenses

These are headers the **server** sends. Missing or misconfigured ones are findings.

### Content Security Policy

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com; object-src 'none'
```

Controls what resources the browser is allowed to load. Primary XSS defense. See [[CSP]] for full breakdown.

**Finding:** Missing CSP, or CSP with `unsafe-inline` / `unsafe-eval` significantly weakens XSS protection.

### Transport Security

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

Forces the browser to use HTTPS for all future requests to this domain (for the duration of `max-age`).

**Finding:** Missing HSTS — downgrade attacks, SSL stripping possible on first visit.

### Clickjacking Protection

```http
X-Frame-Options: DENY
# or modern equivalent:
Content-Security-Policy: frame-ancestors 'none'
```

Prevents the page from being embedded in an `<iframe>`. See [[Browser-Security-Model]].

**Finding:** Missing — clickjacking may be possible. Check if sensitive actions (fund transfer, settings change) are exposed.

### MIME Sniffing

```http
X-Content-Type-Options: nosniff
```

Prevents the browser from guessing the content type if the server's `Content-Type` is wrong. Without it, a server returning `text/plain` that contains HTML can be rendered as HTML in some contexts.

### Information Leakage Headers

Servers often leak tech stack info in response headers. Look for:

```http
Server: Apache/2.4.49 (Ubuntu)       ← version = CVE lookup
X-Powered-By: PHP/7.4.3              ← language + version
X-AspNet-Version: 4.0.30319          ← .NET version
X-Generator: Drupal 8                ← CMS fingerprinting
```

**Finding:** Verbose `Server` / `X-Powered-By` headers. Low severity alone but helps with fingerprinting and CVE research.

### CORS Headers

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

See [[CORS]] for the full attack surface on these.

---

## Security Headers Checklist

When testing a target, check for these in every response:

- [ ] `Content-Security-Policy` present and not trivially bypassable?
- [ ] `Strict-Transport-Security` present?
- [ ] `X-Frame-Options` or `frame-ancestors` in CSP?
- [ ] `X-Content-Type-Options: nosniff`?
- [ ] `Referrer-Policy` set?
- [ ] `Permissions-Policy` set (limits browser features like camera, mic)?
- [ ] `Server` header reveals version info?
- [ ] `X-Powered-By` present?
- [ ] CORS headers allow attacker origins?

> [!TIP]
> [securityheaders.com](https://securityheaders.com) gives a quick grade on security headers for any public domain. Use it for a fast baseline check during recon.

---

*Back to [[HTTP-Deep-Dive]] | Related: [[CORS]] | [[CSP]] | [[Browser-Security-Model]]*

*Sources: OWASP Testing Guide v4, MDN Web Docs, PortSwigger Web Security Academy*
