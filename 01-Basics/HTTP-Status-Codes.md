# HTTP Status Codes

← [[HTTP-Deep-Dive]]

---

Status codes tell you what happened to your request. As a security tester, you read status codes like a detective reads clues.

---

## 2xx — Success

| Code | Name | Security Notes |
|------|------|---------------|
| `200 OK` | Request succeeded | Standard response |
| `201 Created` | Resource was created | POST/PUT success |
| `204 No Content` | Success, no body | Common on DELETE |

---

## 3xx — Redirects

| Code | Name | Security Notes |
|------|------|---------------|
| `301 Moved Permanently` | Permanent redirect | Cached by browsers and CDNs |
| `302 Found` | Temporary redirect | Most common redirect; test for **open redirect** |
| `307 Temporary Redirect` | Redirect preserving method | POST stays POST after redirect |
| `308 Permanent Redirect` | Permanent, method preserved | Like 301 but preserves method |

> [!TIP]
> Open redirect test: Can you control where a `302` redirects? `https://example.com/login?next=https://evil.com` — if it redirects to `evil.com`, that's an open redirect. On its own it's usually low severity, but it can be chained into OAuth token theft or phishing.

---

## 4xx — Client Errors

| Code | Name | Security Notes |
|------|------|---------------|
| `400 Bad Request` | Malformed request | WAF blocking your payload? Or actual bad request? |
| `401 Unauthorized` | Not authenticated | Auth is enforced here; try without token |
| `403 Forbidden` | Authenticated but not authorized | Access control is here — test for bypasses |
| `404 Not Found` | Doesn't exist | Enumeration response; sometimes hidden as 404 but accessible |
| `405 Method Not Allowed` | Wrong HTTP method | Try other methods (PUT, DELETE, PATCH) |
| `408 Request Timeout` | Client too slow | — |
| `413 Payload Too Large` | Body too big | File upload size limits |
| `415 Unsupported Media Type` | Wrong Content-Type | Try changing `Content-Type` to bypass filters |
| `422 Unprocessable Entity` | Validation error | Business logic bypass testing |
| `429 Too Many Requests` | Rate limited | Brute force protection; test for bypass |

> [!NOTE]
> The difference between `401` and `403`:
> - `401` = "You haven't authenticated." Log in first.
> - `403` = "You're authenticated but can't do this." The access control exists — but does it work correctly? Test with other users' IDs.

> [!TIP]
> A `403` on `/admin` doesn't mean you can never access it. Common bypasses:
> - `GET /admin` blocked → try `GET /ADMIN`, `/admin/`, `/admin/.`
> - Add headers: `X-Forwarded-For: 127.0.0.1`, `X-Original-URL: /admin`
> - Try different HTTP methods: `POST /admin`, `HEAD /admin`

---

## 5xx — Server Errors

| Code | Name | Security Notes |
|------|------|---------------|
| `500 Internal Server Error` | App crashed | Potentially verbose error leaking stack traces, DB info, code paths |
| `502 Bad Gateway` | Upstream server error | Proxy/load balancer issue |
| `503 Service Unavailable` | Server overloaded/down | — |
| `504 Gateway Timeout` | Upstream server timeout | Can indicate SSRF time-based probing is working |

> [!WARNING]
> A `500` in response to your input is often a **sign of a vulnerability**. It means the server processed your input in an unexpected way. Dig deeper — is it SQL error? Template error? Deserialization crash?

---

## Using Status Codes During Testing

**Enumeration:** Look for `200` vs `404` to find hidden endpoints. But be careful — some apps return `200` for everything (including 404 pages). Check response **body size** too.

**Authentication testing:**
- Does a protected endpoint return `401` or `403` when accessed without a token?
- Can you get `200` by manipulating the token?

**Access control (IDOR) testing:**
- Access your own resource: `200` ✓
- Access another user's resource: should be `403` or `404`
- Getting `200` = IDOR

**Injection testing:**
- Normal input: `200`
- Injection payload: `500` → something broke → investigate

---

*Back to [[HTTP-Deep-Dive]]*
