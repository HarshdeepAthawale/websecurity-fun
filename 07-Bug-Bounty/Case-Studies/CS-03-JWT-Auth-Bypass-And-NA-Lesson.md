# CS-03: JWT Auth Bypass on Management API (+ Why It Got Closed N/A)

← [[Case-Studies-Overview]]

**Vuln class:** Broken Authentication — JWT Signature Validation Missing  
**Severity:** CVSS 8.6 HIGH (submitted) → **Closed N/A by triager**  
**Program:** Large media company VDP (Intigriti)  
**This case study is as much about the N/A as about the bug itself**

---

## TL;DR

A Spring Boot API had JWT signature validation correctly implemented on its user-facing routes (`/api/*`) but missing on its management routes (`/mgmt/*`). A forged JWT with any payload passed authentication on the admin endpoints. The triager closed it as N/A because: the forged JWT passed auth, but a downstream service outage caused all `/mgmt/` endpoints to return HTTP 500 — so no actual data could be read and impact couldn't be demonstrated.

**The lesson is not "JWT bypass." The lesson is that a valid report requires demonstrated impact, not just a bypassed check.**

---

## The Vulnerability (What Actually Happened)

### How JWT Authentication Works (Quick Recap)

A JWT has three parts: `header.payload.signature`

The signature is computed by the server using a secret key. When you submit a JWT:
1. Server decodes the header and payload (Base64 — anyone can read it)
2. Server recomputes the expected signature using its secret key
3. Server compares the computed signature to the submitted signature
4. **If they don't match → 401 Unauthorized**

If step 4 is skipped — the server accepts any JWT as long as the format is correct — you have a signature bypass.

### The Finding

The application had two route groups in its Spring Security config:

| Route prefix | JWT signature check | Result |
|---|---|---|
| `/api/**` | ✅ Yes — RS256, validates against JWKS | Properly secured |
| `/mgmt/**` | ❌ No — only checks if `Authorization: Bearer X` is present | **Bypassed** |

**Test: Forge a JWT with no valid signature**

```bash
# Forge a JWT: header={"alg":"none"}, payload={"sub":"attacker","role":"admin"}
# This creates a token that any JWT library can decode but no server should accept
FAKE_JWT="eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhdHRhY2tlciIsInJvbGUiOiJhZG1pbiJ9.FAKESIG"

# Test on management endpoint:
curl -H "Authorization: Bearer $FAKE_JWT" https://api.example.com/mgmt/user
# Response: HTTP 500 Internal Server Error {"message": "Internal Server Error"}

# Test the SAME token on the user-facing endpoint:
curl -H "Authorization: Bearer $FAKE_JWT" https://api.example.com/api/code/redemption \
  -H "Content-Type: application/json" -d '{"promotionKey":"TEST","code":"TEST"}'
# Response: HTTP 401 {"error": "401 UNAUTHORIZED \"Signature invalid\""}
```

### The Differential Test Table

This is the critical technique — testing the same inputs across different endpoints to isolate where the check fails:

| Request | `/mgmt/user` | `/api/code/redemption` |
|---|---|---|
| No `Authorization` header | 400 "header not present" | 404 "Promotion not found" |
| `Bearer ` (empty value) | 401 Unauthorized | 401 Unauthorized |
| `Bearer FORGED.JWT.FAKESIG` | **500 (reaches backend)** | **401 "Signature invalid"** |
| `Bearer <valid signed JWT>` | 500 (reaches backend) | 403 "email missing" (auth passes) |

The 500 on the management endpoint with a forged JWT proves authentication was bypassed — the request reached the backend business logic, which then failed for an unrelated reason (downstream service error).

---

## Why It Got Closed N/A

The triager's reasoning:

> "The downstream service error blocks actual exploitation — no unauthorized data access demonstrated."

The triager's logic is correct from a bug bounty perspective. A vulnerability report needs to show:
1. A bypassed check ← ✅ shown
2. **Data or capability gained as a result** ← ❌ NOT shown (500 from all mgmt endpoints)

The downstream service that powers the management endpoints happened to be down. So even with auth bypassed, no management data was readable. The triager's bar for "valid" was: can you actually read or modify something you shouldn't be able to?

The answer was no — not because auth was fixed, but because the backend was broken.

> [!WARNING]
> This is one of the most frustrating situations in bug bounty. You found a real vulnerability. The check is provably absent. But you can't demonstrate data access because a dependent service is down. The triager closes it. **This is part of the game.**

---

## The Lessons

### Lesson 1: Partial bypass ≠ valid report

"The authentication check is bypassed" alone is often not enough. You need to show *what the bypass enables*. For auth bypass findings, the bar is:
- Reading data you shouldn't have access to, OR
- Performing an action you shouldn't be able to perform

Just showing `HTTP 500` when you expected `HTTP 401` is evidence of a bug, but not evidence of exploitable impact.

### Lesson 2: Check the downstream service first

Before submitting a "the forged token reaches the backend" finding, verify the backend is actually up and returning real data. Test with unauthenticated requests to see if the service is returning sensible errors vs. infra errors.

```bash
# Is the service healthy?
curl https://api.example.com/actuator/health
# {"status": "UP"} → service is working, proceed
# {"status": "DOWN", "db": {"status": "DOWN"}} → investigate further first
```

In this case, the Spring Boot actuator showed the app was UP, but the UIM (identity management) downstream service was broken, causing all management endpoints to fail at the business logic layer.

### Lesson 3: The "differential testing" technique is still gold

Even though this report got N/A'd, the technique — **sending the same inputs to multiple endpoints and comparing responses** — is extremely powerful for finding auth inconsistencies. It proves the check exists in one place and doesn't in another.

Use it for:
- Comparing authenticated vs. unauthenticated responses
- Comparing different user roles on the same endpoint
- Comparing different route groups with the same security config

### Lesson 4: Timing matters

If you have a valid auth bypass, submit quickly. Services come back up. When you re-check after submission and the service is healthy, you can update the report with "now getting real data via the bypass." Getting that update in before the triager acts can flip an N/A.

### Lesson 5: The `alg:none` attack

The forged JWT used `alg: none` — a JWT with the algorithm set to "none" has no signature. Older JWT libraries would accept these if the server didn't explicitly reject the `none` algorithm. Modern libraries reject it by default, but if the server is only checking for the presence of a `Bearer ` header (as in this case), the algorithm check never even runs.

Other JWT attacks to know:
- **`alg:none`** — remove signature entirely
- **Algorithm confusion (RS256 → HS256)** — trick a server expecting RS256 into using the public key as an HMAC secret
- **`kid` header injection** — if the `kid` (key ID) parameter is used in a SQL query or file path, SQLi or path traversal

Full reference: [[JWT-Attacks]]

---

## What a Valid JWT Bypass Submission Looks Like

If you find a JWT bypass where the backend IS working, your submission needs:

1. **The forged JWT** — show the exact base64 payload, make it clear there's no valid signature
2. **The 500/200 response with data** — not just "returned 500" but "returned user data / admin functions worked"
3. **The differential comparison** — show the same token is rejected on other endpoints with "Signature invalid"
4. **The full list of affected admin endpoints** — from `/v3/api-docs` or similar — shows the scope of access

---

## Practice This

- **PortSwigger JWT labs** — `labs.portswigger.net/websecurity/jwt` — covers `alg:none`, algorithm confusion, `kid` injection
- **jwt.io** — decode and inspect JWTs in the browser
- [[JWT-Attacks]] in this vault — full reference on JWT attack classes

---

*Related: [[JWT-Attacks]] | [[A07-Auth-Failures]] | [[Reporting-Guide]] | [[Case-Studies-Overview]]*

*Sources: Real Intigriti VDP submission and triager response*
