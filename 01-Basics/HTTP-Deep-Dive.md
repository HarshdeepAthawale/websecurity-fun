# HTTP Deep Dive

[[How-the-Web-Works]] | **Overview** | [[HTTP-Methods]] | [[HTTP-Headers]] | [[HTTP-Cookies-and-Sessions]] | [[HTTP-Status-Codes]]

---

HTTP (HyperText Transfer Protocol) is the application-layer protocol that everything on the web runs on. As a web security person, you need to understand HTTP better than most developers do — because **every web vulnerability is ultimately an HTTP-level problem**.

---

## HTTP is Stateless

HTTP has no memory. Every request is independent — the server doesn't inherently know if request #2 came from the same person as request #1.

This is why **cookies and sessions** exist — they bolt state onto a stateless protocol. And that bolted-on state is a huge source of vulnerabilities.

---

## Anatomy of an HTTP Request

```http
POST /api/login HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 42
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Cookie: tracking=abc123
Authorization: Bearer eyJhbGci...

{"username": "admin", "password": "hunter2"}
```

Breaking it down:

| Part | Example | Notes |
|------|---------|-------|
| Method | `POST` | What action to perform |
| Path | `/api/login` | Which resource |
| HTTP version | `HTTP/1.1` | Still dominant; HTTP/2 and 3 exist |
| Host header | `example.com` | Required in HTTP/1.1 |
| Headers | `Content-Type`, `Cookie`, etc. | Metadata |
| Blank line | | Separates headers from body |
| Body | `{"username":...}` | Only for POST/PUT/PATCH |

---

## Anatomy of an HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=xyz789; HttpOnly; Secure; SameSite=Strict
X-Content-Type-Options: nosniff
Content-Security-Policy: default-src 'self'
Content-Length: 27

{"status": "login success"}
```

| Part | Notes |
|------|-------|
| Status line | `HTTP/1.1 200 OK` |
| Response headers | Instructions to the browser |
| Blank line | Separates headers from body |
| Body | HTML, JSON, file data, etc. |

---

## HTTP Methods

See [[HTTP-Methods]] for a full breakdown. The key ones:

| Method | Purpose | Has Body? | Safe? | Idempotent? |
|--------|---------|-----------|-------|-------------|
| `GET` | Retrieve data | No | Yes | Yes |
| `POST` | Submit data, create resource | Yes | No | No |
| `PUT` | Replace a resource | Yes | No | Yes |
| `PATCH` | Partial update | Yes | No | No |
| `DELETE` | Delete a resource | No | No | Yes |
| `HEAD` | GET but no body in response | No | Yes | Yes |
| `OPTIONS` | Ask server what methods are allowed | No | Yes | Yes |

> [!NOTE]
> **Safe** = doesn't change server state. **Idempotent** = calling it multiple times has the same effect as calling it once.
>
> Security impact: Many apps only protect `POST` routes with CSRF tokens, leaving `PUT`/`DELETE` exposed. Always check all methods.

> [!TIP]
> In Burp, you can change the HTTP method on any request. Changing `GET` to `POST` or vice versa sometimes bypasses auth checks on poorly written apps.

---

## HTTP Headers

See [[HTTP-Headers]] for a complete reference. The most important ones:

### Request Headers (attacker perspective)

| Header | Purpose | Security angle |
|--------|---------|---------------|
| `Host` | Which virtual host | Host header injection, password reset poisoning |
| `Origin` | Where the request came from | CORS bypass |
| `Referer` | Previous page URL | Information leakage, CSRF bypass |
| `User-Agent` | Browser/client identifier | WAF bypass, fingerprinting |
| `X-Forwarded-For` | Real IP behind proxy | IP bypass, rate limit evasion |
| `Content-Type` | Format of request body | Type confusion, filter bypass |
| `Authorization` | Credentials / token | JWT attacks, token leakage |
| `Cookie` | Session data | Session hijacking, CSRF |

### Response Headers (defender perspective)

| Header | Purpose | What happens without it |
|--------|---------|------------------------|
| `Content-Security-Policy` | Restrict sources of JS/CSS | XSS easier to exploit |
| `X-Frame-Options` | Prevent iframes | Clickjacking |
| `X-Content-Type-Options: nosniff` | Don't sniff MIME type | MIME confusion attacks |
| `Strict-Transport-Security` | Force HTTPS | MITM, SSL stripping |
| `Set-Cookie` with `Secure; HttpOnly; SameSite` | Cookie flags | Theft, CSRF |

> [!WARNING]
> Missing security headers are a finding category. They're usually low/informational severity by themselves, but missing `CSP` + XSS = critical.

---

## HTTP Status Codes

See [[HTTP-Status-Codes]] for the full list. The ones that matter most for security:

| Code | Meaning | Security relevance |
|------|---------|-------------------|
| `200 OK` | Success | — |
| `201 Created` | Resource created | — |
| `301/302` | Redirect | Open redirect testing |
| `400 Bad Request` | Malformed request | WAF/filter detection |
| `401 Unauthorized` | Not authenticated | Auth boundary |
| `403 Forbidden` | Authenticated but not authorized | IDOR boundary |
| `404 Not Found` | Doesn't exist | Enumeration |
| `405 Method Not Allowed` | Wrong HTTP method | Method testing |
| `429 Too Many Requests` | Rate limited | Brute force protection check |
| `500 Internal Server Error` | Server crash | Error-based injection, verbose errors |

> [!TIP]
> The difference between `401` and `403` tells you a lot. `401` means "you're not logged in." `403` means "you're logged in but can't do that." A `403` on an admin endpoint means the access control is *there* — but that doesn't mean it's *correct*.

---

## Cookies and Sessions

This is where statelessness gets patched — and where a huge number of vulns live. See [[HTTP-Cookies-and-Sessions]] for the full deep dive.

**Quick summary:**

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict; Path=/; Domain=example.com
```

| Flag | Effect | Missing = |
|------|--------|-----------|
| `HttpOnly` | JS can't read the cookie | XSS can steal it |
| `Secure` | Only sent over HTTPS | Exposed over HTTP |
| `SameSite=Strict` | Not sent on cross-site requests | CSRF possible |
| `SameSite=Lax` | Sent on top-level navigation only | Some CSRF still possible |
| `Path` | Limits which paths send the cookie | Scope creep |
| `Domain` | Which (sub)domains get the cookie | Subdomain cookie theft |

---

## HTTP/2 and HTTP/3

Modern apps use HTTP/2 (over TLS) or HTTP/3 (over QUIC). Key differences from a security perspective:

- HTTP/2 multiplexes requests over one TCP connection — relevant for **HTTP request smuggling**
- Headers are binary in HTTP/2, but Burp translates them for you
- **HTTP Request Smuggling** becomes more interesting with HTTP/2 downgrade scenarios (see [[HTTP-Request-Smuggling]])

For most web app testing, you'll see HTTP/1.1 in Burp regardless of what the app actually uses (Burp normalizes it).

---

## Summary

HTTP is the language you need to be fluent in. Before you can exploit a vulnerability, you need to:
1. See the raw request and response (use Burp Suite)
2. Understand what every header does
3. Know which parts of the request you control as an attacker

---

## What's Next

- [[HTTP-Methods]] — when method abuse leads to bypasses
- [[HTTP-Headers]] — full reference for request and response headers
- [[HTTP-Cookies-and-Sessions]] — the full story on session management
- [[HTTP-Status-Codes]] — what each status code tells you
- [[Web-Architecture]] — now that you know the protocol, let's understand the systems that run on it

---

*Sources: RFC 9110 (HTTP Semantics), MDN Web Docs, PortSwigger Web Security Academy*
