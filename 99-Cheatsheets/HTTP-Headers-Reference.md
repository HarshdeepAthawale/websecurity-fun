# HTTP Headers Quick Reference

Full security context: [[HTTP-Headers]]

---

## Security Response Headers (Server → Client)

| Header | Recommended Value | What It Does |
|--------|-------------------|-------------|
| `Content-Security-Policy` | See [[CSP]] | Restricts resource loading, mitigates XSS |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Forces HTTPS |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Prevents clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Controls Referer header leakage |
| `Permissions-Policy` | `geolocation=(), camera=(), microphone=()` | Restricts browser feature access |
| `Cross-Origin-Opener-Policy` | `same-origin` | Prevents cross-origin window access |
| `Cross-Origin-Embedder-Policy` | `require-corp` | Required for SharedArrayBuffer isolation |
| `Cross-Origin-Resource-Policy` | `same-origin` | Prevents cross-origin resource loads |

---

## Cookie Security Flags

```http
Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Strict; Path=/; Domain=example.com
```

| Flag | Effect |
|------|--------|
| `Secure` | HTTPS only |
| `HttpOnly` | No JS access |
| `SameSite=Strict` | No cross-site requests |
| `SameSite=Lax` | Top-level nav only |
| `SameSite=None; Secure` | Always sent cross-site |

---

## Attack-Relevant Request Headers

| Header | SSRF/Bypass Use |
|--------|----------------|
| `Host` | Host header injection, virtual host routing |
| `X-Forwarded-For` | IP spoofing, rate limit bypass |
| `X-Forwarded-Host` | Cache poisoning, redirect control |
| `X-Original-URL` | 403 bypass on nginx/Apache |
| `X-Rewrite-URL` | 403 bypass on IIS |
| `X-Custom-IP-Authorization` | IP allowlist bypass |
| `True-Client-IP` | IP bypass |
| `Forwarded` | RFC 7239 header — IP bypass |
| `X-Host` | Cache poisoning |

---

## Fingerprinting Headers (Leaks Tech Stack)

| Header | Reveals |
|--------|---------|
| `Server: Apache/2.4.49` | Web server + version |
| `Server: nginx/1.18.0` | Nginx version |
| `X-Powered-By: PHP/7.4.3` | PHP version |
| `X-Powered-By: Express` | Node.js + Express |
| `X-AspNet-Version` | .NET version |
| `X-Generator` | CMS (WordPress, Drupal) |
| `CF-Ray` | Cloudflare CDN |
| `X-Cache` | CDN/proxy cache hit |
| `Via` | Proxy chain |

---

## CORS Headers

| Header | Value | Effect |
|--------|-------|--------|
| `Access-Control-Allow-Origin` | `https://app.example.com` | Allow specific origin |
| `Access-Control-Allow-Origin` | `*` | Allow any origin (no credentials) |
| `Access-Control-Allow-Credentials` | `true` | Allow cookies/auth |
| `Access-Control-Allow-Methods` | `GET, POST, PUT` | Allowed methods |
| `Access-Control-Allow-Headers` | `Content-Type, Authorization` | Allowed request headers |
| `Access-Control-Max-Age` | `86400` | Preflight cache duration |

---

## 403 Bypass Headers

Try these when a path returns 403:

```
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Real-IP: 127.0.0.1
X-Custom-IP-Authorization: 127.0.0.1
```

---

*Full reference: [[HTTP-Headers]] | Related: [[CORS]] | [[CSP]]*
