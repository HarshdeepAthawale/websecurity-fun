# HTTP Cookies and Sessions

← [[HTTP-Deep-Dive]]

---

HTTP is stateless. Cookies and sessions are how the web fakes statefulness. They're also the reason session hijacking, CSRF, and authentication bypass are possible.

---

## What Is a Cookie?

A cookie is a small piece of data the server asks the browser to store and send back on future requests.

Server sets it:
```http
HTTP/1.1 200 OK
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

Browser sends it automatically on every subsequent request to that domain:
```http
GET /dashboard HTTP/1.1
Cookie: session=abc123
```

The browser manages this entirely — the JavaScript on the page doesn't need to do anything.

---

## Cookie Attributes

```
Set-Cookie: name=value; Expires=...; Max-Age=...; Domain=...; Path=...; Secure; HttpOnly; SameSite=...
```

| Attribute | What it does | Security impact |
|-----------|-------------|----------------|
| `Secure` | Only send over HTTPS | Without it: cookie exposed on HTTP, MITM can steal it |
| `HttpOnly` | JS can't read it (`document.cookie` excluded) | Without it: XSS can steal the cookie value |
| `SameSite=Strict` | Never sent in cross-site requests | Without it: CSRF is possible |
| `SameSite=Lax` | Sent on top-level navigations, not sub-requests | Partial CSRF protection |
| `SameSite=None` | Always sent (requires `Secure`) | Intended for cross-site use (payment iframes, etc.) |
| `Domain=example.com` | Sent to example.com and all subdomains | Subdomain can access parent's cookies |
| `Path=/admin` | Only sent for `/admin` paths | Limits scope |
| `Expires` / `Max-Age` | When the cookie expires | Session vs persistent cookie |

> [!WARNING]
> `Domain=example.com` causes the cookie to be sent to ALL subdomains. If `evil.sub.example.com` is compromised or taken over (subdomain takeover), it receives your session cookie.

---

## Sessions vs Cookies

**Cookie-based sessions:** The session ID (a random token) is stored in the cookie. The server maps that token to user data in its database or memory.

```
Cookie: session=xH9kL2mP8nQ...   →   Server looks up this token   →   User: admin, id: 42
```

**Stateless tokens (JWT):** All user data is encoded in the token itself. No server-side lookup needed.

```
Cookie: token=eyJhbGciOiJIUzI1NiJ9...   →   Server decodes and verifies signature   →   {user: admin, id: 42}
```

JWTs have their own vuln class — [[JWT-Attacks]] covers `alg: none`, weak secrets, and kid injection.

---

## Session Lifecycle (and Where It Goes Wrong)

```
1. User logs in
2. Server creates a session, generates a random session ID
3. Server sends: Set-Cookie: session=<random_id>; HttpOnly; Secure
4. User's browser stores it and sends it on every request
5. User logs out
6. Server should invalidate the session ID server-side
```

**Where things go wrong:**

| Issue | Attack |
|-------|--------|
| Session ID is predictable (sequential, timestamp-based) | Session prediction/brute force |
| Session not invalidated on logout | Session fixation / persistent access after logout |
| Session not invalidated after password change | Pre-auth session fixation |
| Long-lived sessions with no idle timeout | Stolen session remains valid for days |
| Session ID in URL (`?sessionid=xxx`) | Referer leakage, shoulder surfing, logs |
| No session rotation after privilege change | Privilege escalation via fixation |

---

## Session Fixation

Attacker tricks the victim into using a session ID that the attacker already knows:

1. Attacker visits the site, gets a session ID (unauthenticated)
2. Attacker crafts a URL: `https://bank.com/login?PHPSESSID=attacker_known_id`
3. Victim clicks the link and logs in using that session ID
4. Attacker now has an authenticated session (same ID, now authed)

**Fix:** Always issue a new session ID after login.

---

## Cookie Theft Techniques

| Technique | How | Defense |
|-----------|-----|---------|
| XSS | JS reads `document.cookie` | `HttpOnly` flag |
| Network MITM | Sniff HTTP traffic | `Secure` flag + HSTS |
| Subdomain cookie theft | Malicious subdomain receives cookie | Careful `Domain` attribute |
| Log/referer leakage | Session ID in URL shows up in logs | Never put tokens in URLs |

> [!NOTE]
> `HttpOnly` stops JS from *reading* the cookie, but an XSS payload can still make authenticated requests — it just can't exfiltrate the token itself. The attacker can act as the victim without knowing the session value.

---

## Testing Checklist

- [ ] Are sessions invalidated on logout (server-side, not just cookie deletion)?
- [ ] Does session ID change after login (prevents fixation)?
- [ ] `HttpOnly` on session cookie?
- [ ] `Secure` on session cookie?
- [ ] `SameSite` attribute set?
- [ ] Is the session ID random/unpredictable (check length, charset, entropy)?
- [ ] Are session IDs in URLs anywhere?
- [ ] What is the session lifetime? Is idle timeout enforced?

---

*Back to [[HTTP-Deep-Dive]] | Related: [[JWT-Attacks]] | [[CSRF]] | [[XSS]]*
