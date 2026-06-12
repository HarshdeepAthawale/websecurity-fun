# HTTP Methods

← [[HTTP-Deep-Dive]]

---

## All Methods and Their Security Relevance

### GET
- Retrieves a resource. Should have **no side effects** (safe + idempotent).
- Parameters are in the URL: `GET /search?q=hello`
- **Security:** URL params end up in logs, browser history, `Referer` header — avoid putting sensitive data (tokens, passwords) in GET params.

### POST
- Submits data in the request body. Used for forms, API calls, file uploads.
- **Security:** The most common method for attack input. SQL injection, XSS, SSRF, etc. typically go through POST bodies.

### PUT
- Replaces an entire resource. `PUT /user/123` replaces user 123's data entirely.
- **Security:** Often less protected than POST. Check if PUT is allowed where it shouldn't be.

### PATCH
- Partial update. `PATCH /user/123` updates only the fields you send.
- **Security:** Same as PUT concerns. Also watch for mass assignment — does PATCH let you update `admin: true`?

### DELETE
- Deletes a resource.
- **Security:** Is this endpoint properly authenticated? Can you delete other users' resources (IDOR)?

### HEAD
- Same as GET but no response body. Used to check if a resource exists without downloading it.
- **Security:** Useful for enumeration (does `/admin` exist?). Sometimes WAFs only check GET/POST.

### OPTIONS
- Returns what methods a server supports for a URL. Browser uses this for **CORS preflight** requests.
- **Security:** Can leak what functionality exists. CORS misconfig often visible here.

### TRACE
- Echoes the request back. Used for debugging.
- **Security:** Should be disabled — was used in `XST` (Cross-Site Tracing) attacks to steal cookies. Mostly irrelevant now.

### CONNECT
- Creates a tunnel (used for HTTPS through proxies).
- **Security:** Proxy SSRF via CONNECT method.

---

## Method Override

Some frameworks support method override — you send a `POST` but include a header or parameter that says "treat this as DELETE":

```http
POST /user/123 HTTP/1.1
X-HTTP-Method-Override: DELETE
```

Or as a query parameter: `POST /user/123?_method=DELETE`

**Security relevance:** WAFs blocking `DELETE` requests won't catch a `POST` with method override. Always test.

---

## Testing Checklist

- [ ] Try all methods on sensitive endpoints (especially PUT, DELETE, PATCH)
- [ ] Check if method override headers are accepted
- [ ] Does OPTIONS expose more than it should?
- [ ] Are `PUT`/`DELETE` protected by auth when `POST`/`GET` are?

---

*Back to [[HTTP-Deep-Dive]]*
