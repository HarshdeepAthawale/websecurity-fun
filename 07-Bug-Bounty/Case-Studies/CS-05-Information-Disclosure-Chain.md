# CS-05: Information Disclosure → Recon Escalation Chain

← [[Case-Studies-Overview]]

**Vuln class:** Information Disclosure (Multiple)  
**Severity:** P3 (CVSS 5.3 MEDIUM) — ASP.NET verbose errors / P4 — Git hash + version disclosure  
**Program:** Large media company VDP (Intigriti)  
**The lesson:** Info disclosure isn't "just low." It's reconnaissance that enables targeted attacks.

---

## TL;DR

An accreditation portal ran in production with `customErrors mode="Off"` — the ASP.NET setting that makes the server dump full exception stack traces to any visitor. No authentication needed. Each error response leaked: the development vendor name, absolute Windows file paths, controller class names, exact line numbers, and the framework version string. Separately, health check endpoints on multiple internal APIs leaked git commit hashes and build versions.

None of these are "game over" findings on their own. But they're the intelligence that makes every subsequent attack smarter.

---

## The Core Finding: ASP.NET Verbose Errors

**How it was found:**

During subdomain enumeration, an accreditation portal was discovered (`akkreditierung.company.com`). Testing standard paths:

```bash
# Request any protected path (no authentication)
curl -s https://akkreditierung.company.com/admin
```

**Response: HTTP 500 Internal Server Error** — but with a full "yellow screen of death":

```
Server Error in '/' Application.
Object reference not set to an instance of an object.

Stack Trace:
at AccreditationPortal.Web.Controllers.BaseController.CheckUserTokenExp
   at C:\VENDORNAME\PROJECTS\AccreditationPortalFrontend\AccreditationPortal.Web\Controllers\BaseController.cs:78

at AccreditationPortal.Web.Controllers.BaseController.Initialize
   at C:\VENDORNAME\PROJECTS\AccreditationPortalFrontend\AccreditationPortal.Web\Controllers\BaseController.cs:34

Version Information: .NET Framework Version:4.0.30319; ASP.NET Version:4.8.4797.0
```

**What this reveals:**

| Information Leaked | Attacker Use |
|--------------------|--------------|
| Vendor name (AxessAG) | Search CVE databases for AxessAG AccreditationPortal vulnerabilities |
| Server OS = Windows | Target Windows-specific exploit paths |
| Absolute file path (`C:\VENDORNAME\PROJECTS\...`) | LFI/path traversal path construction |
| Controller class name | Source code structure mapping |
| Method name (`CheckUserTokenExp` at line 78) | Token validation logic is at this exact location — targeted attack |
| Framework version (ASP.NET 4.8.4797.0 / .NET 4.0.30319) | Look up exact CVEs for this patch level |

**The NullReferenceException in `CheckUserTokenExp`** is particularly interesting — it suggests the token validation runs before the null check, meaning it might be reachable with a crafted request that bypasses token validation.

**Root cause:**

In ASP.NET, the `customErrors` setting in `Web.config` controls error page behavior:
```xml
<!-- What was on production: -->
<customErrors mode="Off" />

<!-- What it should be: -->
<customErrors mode="RemoteOnly" />
<!-- or -->
<customErrors mode="On" />
```

`mode="Off"` sends full error details to every client. `mode="RemoteOnly"` only shows details to `localhost`, sending generic 500 pages to everyone else. This is one of the most basic ASP.NET hardening steps — and it was missing on production.

---

## Additional Findings: Health Endpoint Disclosure

Many modern web frameworks expose a `/health` endpoint for uptime monitoring. These often leak more than intended:

```bash
curl https://api.player.example.com/health
# Response:
{"status":"UP","git":"f35ea25f741414335a20646a282565d7332ad6db","_meta":{"serverTime":"2026-06-05T08:05:32.910Z"}}

curl https://api.b2b-portal.example.com/api
# Response:
{"status":"OK","version":"1.8.0","startDateTime":"...","uptime":"..."}
```

**What these expose:**

- **Git commit hash** — can be used to look for the exact version of open-source frameworks being used, find public CVEs for this commit, or (if the repo is public/leakable) identify code changes
- **Version number** — direct CVE lookup for the exact version
- **Server time** — baseline for timing attacks, TOTP window calculations

Individually, these are P4 (low severity, informational). But they're fast to find and always worth noting.

---

## Bonus: User Enumeration on the Same Target

The same accreditation portal also had user enumeration via its forgot password flow:

```bash
# Forgot password with non-existent username:
curl -s -X POST https://akkreditierung.example.com/AccountPublic/ForgotPassword \
  -d '{"Username":"nonexistent"}'
# Response: {"validation":[{"PropertyName":"Username","ErrorMessage":"Username existiert nicht!"}]}
# (German: "Username does not exist!")

# Login error message:
curl -s -X POST https://akkreditierung.example.com/AccountPublic/Login \
  -d '{"Username":"x","Password":"x"}'
# Response: {"validation":[{"PropertyName":"Password","ErrorMessage":"Benutzer nicht gefunden oder Passwort ist ungültig."}]}
# (German: "User not found or password is invalid.")
```

The forgot password explicitly says "username doesn't exist" vs login which gives a merged message. This is **user enumeration** — you can check whether a username is registered without authenticating. P4 on its own, combined with verbose errors in the same report.

---

## Why Info Disclosure Matters: The Attack Chain

Here's why these "low severity" findings matter in context:

```
Step 1: Verbose errors → identify vendor as "AxessAG"
Step 2: Google "AxessAG AccreditationPortal CVE" → find known vulnerabilities
Step 3: Verbose errors → exact framework version (ASP.NET 4.8.4797.0)
Step 4: CVE lookup for this exact build → find unpatched vulnerabilities
Step 5: Stack trace → "CheckUserTokenExp" at BaseController.cs:78
Step 6: Understand the auth flow → craft attack targeting this specific method
Step 7: User enumeration → identify valid usernames
Step 8: Targeted credential attack → now with actual usernames and vendor-specific exploit
```

Each piece of information feeds the next. A solo P4 finding becomes a full attack chain when combined.

---

## CVSS Breakdown

**ASP.NET verbose errors:**
```
AV:N  - Internet-accessible endpoint
AC:L  - GET request to any path
PR:N  - No authentication
UI:N  - No user interaction
S:U   - Scoped to this application
C:L   - Limited information: paths, versions, stack trace (not credentials)
I:N   - Read-only
A:N   - No availability impact

Score: 5.3 MEDIUM
```

**Git hash / version disclosure:**
```
AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N → 5.3 MEDIUM (or P4 in VRT context)
```

**VRT:** `Information Disclosure → Technical Information Disclosure → Verbose Error Messages` → **P3–P4**

---

## How to Find These Fast

```bash
# ASP.NET error disclosure — trigger any 500
curl https://target.com/admin
curl https://target.com/nonexistent/path/that/should/not/exist

# Health endpoints (common paths)
for path in /health /actuator/health /status /_health /ping /api /api/health /api/version; do
  echo -n "$path: "
  curl -s -o /dev/null -w "%{http_code}" "https://target.com$path"
  echo
done

# Version disclosure in response headers
curl -I https://target.com | grep -i "x-powered-by\|server\|x-generator\|x-aspnet"
```

**What to look for in error pages:**
- Technology stack (`x-powered-by`, server header, stack trace)
- Absolute file system paths
- Database connection strings (rare but happens)
- Internal hostnames

---

## The Combined Report

These were submitted as a single report because they all relate to information disclosure on the same target. Combining related low-severity findings into one report is good practice:

- Three P4 findings submitted separately → three P4 payouts (if any)
- Three P4 findings combined into one report with a clear attack chain → triager may rate P3 for the combined impact narrative

---

## Practice This

- **PortSwigger:** Information disclosure labs (specifically the ones about tech stack leakage)
- **Any ASP.NET app:** Try `/admin`, `/Home/Index`, and observe error handling behavior
- **Health endpoints:** Next time you're recon-ing a target, include a sweep of health endpoint paths

---

*Related: [[Fingerprinting]] | [[A05-Security-Misconfiguration]] | [[A09-Logging-Failures]] | [[Case-Studies-Overview]]*

*Sources: Real Intigriti VDP submission*
