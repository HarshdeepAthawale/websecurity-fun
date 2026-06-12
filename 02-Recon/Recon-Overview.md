# Recon Overview

**Overview** | [[Subdomain-Enumeration]] | [[Fingerprinting]] | [[Directory-Bruteforce]] | [[JS-Recon]] | [[OSINT]]

---

Recon is the phase where you map out everything about a target before touching it offensively. Good recon is what separates people who find bugs from people who spin their wheels on the same endpoints everyone else tested.

The goal: **build the most complete picture of the attack surface before you send a single malicious payload.**

---

## The Recon Mindset

You're asking three questions:

1. **What exists?** — subdomains, endpoints, parameters, files
2. **What technology is it running?** — framework, language, CDN, WAF, server
3. **What has changed or been exposed?** — old endpoints, leaked secrets, historical data

The more thoroughly you answer these, the more unique your attack surface becomes. Most bug hunters hit the obvious stuff. Recon gets you to the stuff they missed.

---

## Recon Phases

```
1. Scope clarification
       ↓
2. Asset discovery (subdomains, IPs, ASNs)
       ↓
3. Live host identification
       ↓
4. Tech stack fingerprinting
       ↓
5. Surface mapping (endpoints, parameters, JS files)
       ↓
6. OSINT (historical data, leaks, job posts, GitHub)
       ↓
7. Attack surface prioritization
```

---

## Phase 1 — Scope Clarification

Before you enumerate anything, understand what you're allowed to test.

On bug bounty programs, scope is defined in the program policy:
- **In scope:** `*.example.com`, `api.example.com`
- **Out of scope:** `status.example.com`, third-party services

**Golden rule:** If it's out of scope, don't touch it. Not even "just a quick look."

Things to note from the scope:
- Wildcard scopes (`*.example.com`) = all subdomains are in
- Specific asset scopes = only those exact hosts
- Any excluded vulnerability classes (e.g., "rate limiting not accepted")

---

## Phase 2 — Asset Discovery

Start wide. Find every asset associated with the target.

### Subdomain Enumeration

The most important recon step for most targets. See [[Subdomain-Enumeration]] for the full methodology.

Quick start:
```bash
subfinder -d example.com -o subdomains.txt
```

### IP Ranges / ASN

Who owns the IP ranges associated with this org? Useful for finding unlisted assets.

```bash
# Find ASN from org name
curl -s "https://api.bgpview.io/search?query_term=example+corp" | jq '.data.asns'

# Get IP prefixes for an ASN
curl -s "https://api.bgpview.io/asn/12345/prefixes" | jq '.data.ipv4_prefixes[].prefix'
```

### Certificate Transparency

SSL certificates are logged publicly. Every subdomain that ever got a cert is findable:

```bash
# crt.sh (web)
https://crt.sh/?q=%.example.com

# Command line
curl -s "https://crt.sh/?q=%.example.com&output=json" | jq '.[].name_value' | sort -u
```

---

## Phase 3 — Live Host Identification

Not every subdomain resolves to a live host. Filter them:

```bash
# Check which subdomains are live
cat subdomains.txt | dnsx -silent | httpx -silent -o live_hosts.txt
```

Also look for:
- **Interesting status codes** — 403 might be hiding something; 401 = auth required
- **Different technologies** — same org, different stacks on different subdomains
- **Staging/dev environments** — `dev.`, `staging.`, `test.`, `beta.` — often less hardened

---

## Phase 4 — Tech Stack Fingerprinting

See [[Fingerprinting]] for the full breakdown. Quick hits:

- Response headers: `Server`, `X-Powered-By`, `X-Generator`
- Cookies: `PHPSESSID` = PHP, `JSESSIONID` = Java, `ASP.NET_SessionId` = ASP.NET
- Error pages: framework-specific error messages
- File extensions in URLs: `.php`, `.aspx`, `.jsp`
- HTML comments and meta tags

---

## Phase 5 — Surface Mapping

### Directory/Endpoint Bruteforce

Find hidden paths that aren't linked from the UI. See [[Directory-Bruteforce]].

```bash
ffuf -u https://example.com/FUZZ -w /path/to/wordlist.txt -mc 200,301,302,403
```

### Parameter Discovery

Find hidden GET/POST parameters:

```bash
# Arjun — parameter brute-forcer
arjun -u https://example.com/api/user
```

### JavaScript Analysis

Modern apps put a lot in their JS bundles — API endpoints, internal paths, tokens. See [[JS-Recon]].

```bash
# Extract URLs from JS files
cat js_files.txt | grep -oP "https?://[^'\")\s]+"
```

---

## Phase 6 — OSINT

See [[OSINT]] for the full breakdown. Key sources:

- **Wayback Machine** — historical URLs, old endpoints, removed pages
- **GitHub** — leaked secrets, internal tools, old code
- **Shodan** — internet-facing services on the org's IP ranges
- **Google dorks** — find exposed files, admin panels, sensitive data
- **Job postings** — leak the tech stack ("experience with X required")

---

## Phase 7 — Prioritization

Not all attack surface is equal. Prioritize:

1. **Authenticated endpoints** with complex business logic (payments, admin panels, user management)
2. **File upload functionality**
3. **API endpoints** — especially ones that take user IDs as parameters (IDOR)
4. **OAuth / SSO flows**
5. **Old/forgotten subdomains** — less maintained, less hardened
6. **Third-party integrations** — webhooks, API callbacks

---

## Recon Output Structure

Keep your recon organized:

```
recon/
├── subdomains.txt          ← all discovered subdomains
├── live_hosts.txt          ← live subdomains with status codes
├── endpoints.txt           ← discovered endpoints from bruteforce + crawl
├── js_endpoints.txt        ← endpoints extracted from JS
├── secrets.txt             ← anything that looks like a key/token
├── fingerprints.md         ← tech stack notes per host
└── interesting.md          ← things to come back to, odd behaviors
```

---

## What's Next

- [[Subdomain-Enumeration]] — tools and techniques for finding all subdomains
- [[Fingerprinting]] — identifying the tech stack
- [[Directory-Bruteforce]] — finding hidden endpoints
- [[JS-Recon]] — mining JavaScript files for endpoints and secrets
- [[OSINT]] — historical data, Shodan, Google dorks

---

*Sources: OWASP Testing Guide v4, HowToHunt, Bug Bounty Bootcamp*
