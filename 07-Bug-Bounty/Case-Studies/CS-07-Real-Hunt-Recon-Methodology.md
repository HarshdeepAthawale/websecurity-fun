# CS-07: How a Real Bug Bounty Hunt Actually Works

← [[Case-Studies-Overview]]

**Type:** Methodology Case Study  
**Program:** Large media company VDP — ~5900 domains in scope  
**Outcome:** 6 confirmed findings (P1–P3) across ~10 days

---

## TL;DR

This is a breakdown of how a real bug bounty hunt was run — not a sanitized tutorial, but the actual decision-making, dead ends, pivots, and prioritization. Reading this alongside a lab guide is the difference between knowing how to use a tool and knowing how to hunt.

---

## Phase 1: Understanding the Scope

Before running any tool, read the program rules carefully:

**What the rules said:**
- ~5900 domains in scope (from a publicly linked list)
- No automated scanning beyond 5 requests/second
- Out of scope: cache poisoning, CORS on non-sensitive endpoints, missing headers, info disclosure without sensitive data
- Platform: Intigriti (not HackerOne) — different triage team, different culture

**What that tells you:**
1. 5900 domains = massive attack surface. Don't try to touch everything — prioritize.
2. 5 req/sec limit means no aggressive brute-forcing without throttling your tools
3. "Info disclosure without sensitive data" is out of scope — but info disclosure WITH sensitive data (keys, PII, stack traces that expose vendor identity) is in scope
4. Intigriti programs tend to be stricter on CVSS than HackerOne — factor this into your severity estimates

**Key scope insight:** Wide scope programs reward **breadth-first recon followed by depth on interesting targets**, not slow exhaustive testing of one app.

---

## Phase 2: Subdomain Enumeration and Triage

**Tools used:**
- `subfinder`, `assetfinder` for passive enumeration
- `chaos` project data for VDP scope
- `httpx` to probe all discovered subdomains for live hosts

**The output:** ~200+ live hosts from the 5900 scope.

**The first triage:** Not all 200+ hosts are worth testing. Prioritize based on:

| Signal | Priority |
|--------|----------|
| Custom apps (not off-the-shelf SaaS) | HIGH — custom code = custom bugs |
| API endpoints (especially `/api/*`, `/graphql`, `/v1/*`) | HIGH — where data lives |
| Staging/QA environments | HIGH — often misconfigured, less monitored |
| Admin/management panels | HIGH — high impact if access is possible |
| Third-party platforms (Zendesk, Salesforce, etc.) | MEDIUM — out of scope usually |
| Static marketing sites | LOW — unlikely to have auth bugs |
| Generic CDN assets | SKIP |

**First scans after initial triage:**
```bash
# Check for interesting headers on all live hosts
cat live-hosts.txt | httpx -silent -title -status-code -tech-detect -H "Origin: https://evil.com" \
  | grep -E "(graphql|admin|api|staging|directus|artifactory|jenkins)"
```

---

## Phase 3: Technology Fingerprinting

For each interesting host, identify the tech stack before testing anything:

```bash
# Response headers
curl -sI https://target.example.com | grep -i "x-powered-by\|server\|x-generator\|via"

# Common framework paths
curl -s https://target.example.com/actuator/health   # Spring Boot
curl -s https://target.example.com/v3/api-docs       # OpenAPI / Swagger
curl -s https://target.example.com/graphql           # GraphQL endpoint
curl -s https://target.example.com/robots.txt        # Often reveals structure
curl -s https://target.example.com/.well-known/      # Auth discovery
```

**What was found at this stage:**
- `x-powered-by: Directus` → immediately test GraphQL introspection
- JFrog Artifactory on `artifactory.` subdomain → test anonymous API access
- Spring Boot actuator at `/actuator/health` → check for `/v3/api-docs` next
- SvelteKit/Next.js SPA → fetch JS bundles and mine for credentials, endpoints, auth structure

> [!TIP]
> Technology fingerprinting is fast. You can sweep 50 hosts in minutes. The payoff is knowing exactly which tests to run next without wasting time on apps you don't understand.

---

## Phase 4: JS Bundle Mining

For every SPA (React, Next.js, SvelteKit, Angular, Vue), the JavaScript bundles are production artifacts that every visitor downloads. They contain:

```bash
# Find and download JS bundles
# In browser DevTools: Sources tab → right-click each .js file → Save

# Search for interesting patterns
grep -r "api\|endpoint\|token\|secret\|key\|password\|admin" *.js | grep -v "//.*api"
grep -r "Bearer\|Authorization\|X-API-Key" *.js
grep -r "magicToken\|dev-token\|test-\|staging-" *.js  # dev fixtures in prod
grep -r "routes:\|routeList:\|pathList:" *.js  # route tables
```

**What was found from JS bundles:**
- Admin route tree (`/admin`, `/admin/players`, `/admin/teams`, etc.)
- Auth flow structure (magic link → PIN → session cookie)
- Hardcoded player credentials (see [[CS-04-Brute-Force-And-Hardcoded-Creds]])
- OAuth client IDs and tenant IDs (used to understand auth providers)
- API base URLs and endpoint paths

---

## Phase 5: Targeted Testing

Once you know the tech stack and have the API surface mapped, go deep on each interesting target.

**Directus CMS pattern:**
1. See `x-powered-by: Directus`
2. Test: `POST /graphql` with introspection query
3. If open → enumerate collections → extract data
4. Check `/users` (system collection) for admin account exposure
5. **Check all other Directus instances for the same pattern** (see [[CS-01-GraphQL-Unauthenticated]])

**Spring Boot API pattern:**
1. Confirm tech via `actuator/health`
2. Get full API surface from `/v3/api-docs` or `/swagger-ui.html`
3. Map authenticated vs unauthenticated endpoints
4. Test JWT validation: `alg:none`, forged signatures, algorithm confusion
5. Compare auth behavior across different route groups (user routes vs admin routes)

**Authentication pattern:**
1. Find the login mechanism
2. Count the entropy of auth tokens/PINs
3. Test rate limiting with parallel requests
4. Check for lockout policies
5. Test token expiry

---

## Phase 6: Dead End Management

Most targets don't yield findings. Tracking dead ends prevents wasting time re-testing the same things.

**Dead ends from this hunt (what was investigated and closed):**

| Target | Why Closed |
|--------|------------|
| Account management API | JWT validation solid — alg:none, HS256 confusion, bad RS256 sig all rejected properly |
| Contest admin API | 124 API routes extracted from bundle, all returned 401 — no bypass found |
| Analytics API | Azure AD protected, Akamai WAF blocking path traversal attempts |
| Docker Registry | 401 on all endpoints, default creds didn't work |
| Various basic auth portals | Default credentials didn't work, not worth brute-forcing |
| Management API JWT bypass | Technically confirmed bypass but N/A'd because backend service was down (see [[CS-03-JWT-Auth-Bypass-And-NA-Lesson]]) |

**The habit:** Mark dead ends as closed. Otherwise you circle back and waste time. A simple text file tracking `DEAD: target — reason` is enough.

---

## Phase 7: Pattern Replication

**The most efficient move in bug bounty:** when you find a bug class, immediately check if the same bug exists elsewhere.

**What happened here:**
1. Found Directus with open introspection on one staging CMS (app configuration)
2. **Immediately searched for all other Directus instances in scope**
3. Found a second instance (competition platform CMS) with the same misconfiguration — different data, different impact, second report

Same pattern:
- Found Artifactory anon access → checked if any related Artifactory instances exist → found a second one but weaker (packages not downloadable) — not worth a separate report
- Found git hash disclosure on one API health endpoint → noted on all health endpoints → found it on multiple targets — included in same report

---

## Phase 8: Severity Calibration Before Submitting

Before writing a report, estimate whether it's worth submitting. Wrong severity estimate wastes your time and the triager's.

**Quick severity checklist:**
```
Is there demonstrated data access? (not just "could access")         ← Yes → P2 or higher
Is authentication completely bypassed?                               ← Yes → P1 or P2
Is there real user PII exposed? (names, emails, phones)              ← Yes → P2
Is there credential exposure? (API keys, tokens, passwords)          ← Yes → P1 or P2
Is it a staging environment with production data?                    ← Yes → same severity as prod
Is exploitation completely mitigated by another control?             ← Maybe → P4 or P5
Is this purely informational? (no impact statement possible)         ← P4 or skip
```

---

## Phase 9: Writing the Report

After finding and validating, write the report. Good reports:

1. **Title:** includes vulnerability class, target, and what the impact is
2. **CVSS:** calculate it yourself before submitting (triagers appreciate accurate self-assessment)
3. **Steps to Reproduce:** exact curl commands that anyone can run, with expected output
4. **Evidence:** screenshots, response bodies, evidence files
5. **Impact:** state what an attacker could ACTUALLY DO with this access — not just "this could be exploited"
6. **Fix:** concrete recommendation, not just "please fix this"

Full guide: [[Reporting-Guide]]

---

## What Good Looks Like: Findings per Hour

After 10 days of hunting with a realistic time investment:
- 1 Critical (supply chain / Artifactory)
- 4 High (GraphQL ×2, PIN brute force, JWT bypass attempt)
- 1 Medium (ASP.NET verbose errors)
- 1 Low (CORS)

**Rough ratio:** ~1 submittable finding per 1-2 hours of focused work on the right target. Most time is spent on dead ends.

---

## Practical Takeaways

1. **Read the scope list, not just the program description** — the actual in-scope list often has 10× more targets than the "main" domain shown on the program page
2. **Fingerprint first, test second** — knowing the tech stack takes 5 minutes and saves hours
3. **JS bundles are intelligence** — always download and read them for SPAs
4. **Staging environments are often misconfigured** — they're high-value targets that developers don't monitor as closely
5. **Pattern replication doubles your findings** — find a bug class once, check everywhere
6. **Track dead ends** — you will circle back if you don't
7. **Calibrate severity before you write** — a P4 report takes as long to write as a P2, so spend the time on things worth writing
8. **Wide scope programs > narrow scope** — more targets = more opportunities

---

*Related: [[Recon-Overview]] | [[Subdomain-Enumeration]] | [[JS-Recon]] | [[Fingerprinting]] | [[Reporting-Guide]] | [[Case-Studies-Overview]]*

*Sources: Real multi-session bug bounty hunt, distilled from session notes*
