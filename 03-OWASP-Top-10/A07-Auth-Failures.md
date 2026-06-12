# A07 — Identification and Authentication Failures

← [[OWASP-Overview]]

---

Auth failures cover anything that lets an attacker gain unauthorized access to an account or bypass authentication entirely. These are high-severity findings — compromising authentication often means full account takeover.

---

## What Falls Under This Category

- Brute force attacks (no rate limiting, no lockout)
- Weak credentials (default passwords, guessable passwords)
- Broken password reset flows
- Session management issues
- JWT vulnerabilities
- Credential stuffing exposure
- Missing MFA

---

## Brute Force and Credential Stuffing

If there's no rate limiting or account lockout on login endpoints, an attacker can try thousands of passwords.

**Credential stuffing** is worse — attacker takes leaked username/password pairs from other breaches and tries them on your app. If users reuse passwords, this works.

**Testing:**
```bash
# Check for rate limiting on login
ffuf -u https://example.com/login \
  -X POST \
  -d "username=admin&password=FUZZ" \
  -w passwords.txt \
  -mc 200 \
  -H "Content-Type: application/x-www-form-urlencoded"
```

**Signs of no rate limiting:** No change in response after 50+ failed attempts, no `429` status, no CAPTCHA.

---

## Broken Password Reset

Password reset flows are goldmines for auth bugs. Common issues:

### Predictable Reset Tokens

```
# Token that's just a timestamp
/reset?token=1686821234

# Token that's the user's email MD5
/reset?token=5f4dcc3b5aa765d61d8327deb882cf99

# Sequential token
/reset?token=00001234
```

**Test:** Request a reset, look at the token format. Is it random? High entropy? If not — it's guessable.

### Token Not Invalidated After Use

```
# Use the token, reset your password
# Try the same token again → still works
```

### Long Token Expiry

A reset token valid for 7 days (or indefinitely) is a persistent attack window.

### Host Header Injection in Reset Email

See [[HTTP-Headers]]. If the reset email is built using the `Host` header without validation:

```http
POST /forgot-password HTTP/1.1
Host: attacker.com
...
email=victim@example.com
```

The victim receives a reset link pointing to `attacker.com` — attacker captures the token.

### Response Leaking the Token

Sometimes the token is returned in the API response body:
```json
{"message": "Reset email sent", "token": "abc123def456"}
```

---

## Username Enumeration

Does the app tell you whether a username/email exists?

```
"No account found with that email"     ← user doesn't exist
"Password reset email sent"            ← user exists
```

vs the secure response:
```
"If an account with that email exists, a reset email has been sent"   ← can't enumerate
```

Username enumeration enables targeted attacks. It's usually a low/info finding but important context for further exploitation.

**Also check:** Login error messages. `"Invalid username"` vs `"Invalid password"` reveals whether the username is valid.

---

## JWT Vulnerabilities

JWTs are tokens with three base64-encoded parts: `header.payload.signature`

### Algorithm Confusion: `alg: none`

```json
// Original header:
{"alg": "HS256", "typ": "JWT"}

// Attacker modifies to:
{"alg": "none", "typ": "JWT"}
```

Some libraries accept `alg: none` and skip signature verification entirely. Attacker can forge any payload.

### Weak Secret

If the HMAC secret is weak, you can crack it:

```bash
hashcat -a 0 -m 16500 eyJ...token...here /path/to/wordlist.txt
```

A cracked secret = forge any token with any payload (including `"role": "admin"`).

### Algorithm Confusion: RS256 → HS256

If the server uses RS256 (asymmetric), the public key is public. If you change the algorithm to HS256 (symmetric) and sign with the public key as the HMAC secret, some libraries will verify it correctly.

Deep dive: [[JWT-Attacks]]

---

## Session Management Issues

See [[HTTP-Cookies-and-Sessions]] for full detail. Quick checklist:

- [ ] Session invalidated on logout (server-side)?
- [ ] New session ID issued after login (prevents fixation)?
- [ ] `HttpOnly`, `Secure`, `SameSite` flags on session cookie?
- [ ] Session expires after inactivity?
- [ ] Session ID random and unpredictable?

---

## Default Credentials

Admin panels, management interfaces, and IoT devices often ship with default credentials.

**Test:** Try `admin:admin`, `admin:password`, `admin:123456`, `root:root` on any discovered admin interface.

Reference: [Default Credentials Cheat Sheet](https://github.com/ihebski/DefaultCreds-cheat-sheet)

---

## Testing Auth Flows

1. **Registration:** Can you register with an existing username/email? What are password requirements?
2. **Login:** Rate limited? Username/password enumeration via different error messages?
3. **Password reset:** Token format, expiry, host header injection, invalidation after use
4. **Session:** Logout actually invalidates the session?
5. **MFA:** Can it be bypassed? Response manipulation (`"mfa_required": false`)?
6. **Remember me:** How long? What token is used?

---

*Related: [[JWT-Attacks]] | [[HTTP-Cookies-and-Sessions]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, OWASP Testing Guide v4, PortSwigger Web Security Academy*
