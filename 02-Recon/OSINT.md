# OSINT

← [[Recon-Overview]]

---

OSINT (Open Source Intelligence) is finding information about a target using publicly available sources — without touching the target directly. It's entirely passive and often reveals things active scanning never would.

---

## What You're Looking For

- **Infrastructure:** IPs, hostnames, services running on non-standard ports
- **Technology:** what stack they run, what cloud provider
- **Historical data:** old URLs, removed pages, past configs
- **Leaked secrets:** API keys, credentials in GitHub commits, pastes
- **Organizational data:** employee names, email formats, org structure (useful for social engineering context and username guessing)

---

## Wayback Machine / Web Archives

The Wayback Machine (archive.org) has snapshots of pages going back years. Developers remove sensitive pages, change configs, delete endpoints — but archived versions remain.

```bash
# All URLs ever crawled for a domain
waybackurls example.com | sort -u > wayback_urls.txt

# Or with gau (GetAllURLs) — hits Wayback + OTX + Common Crawl
gau --threads 5 example.com | sort -u > gau_urls.txt
```

What to look for:
- Old API versions (`/api/v0/`, `/api/internal/`)
- Backup files that were briefly exposed (`/backup.zip`, `/db.sql`)
- Old admin panels
- JS files from years ago that may have had secrets
- Removed pages with interesting functionality

```bash
# Filter for interesting paths
cat wayback_urls.txt | grep -iE "\.(php|aspx|js|json|xml|yaml|env|bak|sql|zip|tar|gz)$"
cat wayback_urls.txt | grep -iE "(admin|backup|config|debug|internal|test|dev|staging)"
```

---

## Google Dorks

Google indexes public web content. Dorks use advanced search operators to find specific exposed content.

```
site:example.com               → only results from example.com
inurl:/admin                   → URL contains /admin
intitle:"Index of"             → directory listing pages
filetype:pdf                   → only PDF files
```

### Useful Dork Combinations

```
# Find login pages
site:example.com inurl:login OR inurl:signin OR inurl:admin

# Exposed sensitive files
site:example.com ext:env OR ext:log OR ext:sql OR ext:bak

# Directory listings
site:example.com intitle:"Index of /"

# Error messages leaking stack traces
site:example.com "Fatal error" OR "stack trace" OR "SQL syntax"

# Subdomains not shown in normal search
site:*.example.com -www

# Exposed API docs
site:example.com inurl:swagger OR inurl:api-docs OR inurl:openapi

# Config files
site:example.com ext:xml | ext:conf | ext:cnf | ext:reg | ext:inf | ext:rdp | ext:cfg | ext:txt | ext:ora | ext:ini
```

> [!TIP]
> [GoogD0rker](https://github.com/ZephrFish/GoogD0rker) and the [Google Hacking Database (GHDB)](https://www.exploit-db.com/google-hacking-database) have thousands of ready-to-use dorks organized by category.

---

## Shodan

Shodan is a search engine for internet-connected devices. It scans the internet and indexes what's running on every IP.

```
# Search for the org's IPs
org:"Example Corp"

# Find services on specific ports
hostname:example.com port:8080
hostname:example.com port:8443
hostname:example.com port:27017   ← MongoDB

# Find by header content
http.title:"Admin" org:"Example Corp"
http.favicon.hash:-297069493      ← specific favicon hash

# Exposed admin panels
http.title:"phpMyAdmin" org:"Example Corp"
http.title:"Kibana" org:"Example Corp"
```

Shodan CLI:
```bash
shodan search --fields ip_str,port,org,hostnames "org:example"
shodan host 93.184.216.34
```

What Shodan finds that normal recon misses:
- Dev/test servers on non-standard ports
- Exposed databases (MongoDB, Elasticsearch, Redis)
- Internal services accidentally internet-facing
- Old/legacy systems with known CVEs
- Default credential screens

---

## GitHub Dorking

Developers accidentally commit secrets to GitHub constantly. This is one of the highest-yield OSINT techniques.

### Manual Search Operators

```
# Search for org's code
org:examplecorp

# Specific secrets
org:examplecorp "api_key"
org:examplecorp "password" filename:.env
org:examplecorp filename:config.json "password"
org:examplecorp "BEGIN RSA PRIVATE KEY"

# Recent commits mentioning the domain
"example.com" "password"
"@example.com" password
"example.com" api_key
```

### Automated Tools

```bash
# truffleHog — searches git history for secrets
trufflehog github --org=examplecorp --only-verified

# gitleaks
gitleaks detect --repo-url https://github.com/examplecorp/somerepo

# gitrob — maps an org's GitHub presence
gitrob analyze examplecorp
```

### What to Look For

- `.env` files committed
- API keys, AWS credentials
- Database connection strings
- Private keys (`-----BEGIN RSA PRIVATE KEY-----`)
- Hardcoded passwords in config files
- Internal endpoints and service names from code comments

---

## Certificate Transparency (for More Than Subdomains)

Beyond subdomains, certs can reveal:
- Email addresses used for cert registration
- Internal hostnames that appear in SANs (Subject Alternative Names)
- Certificate issuance patterns that suggest new services

---

## Job Postings

Companies advertise their tech stack in job descriptions. This is free recon.

```
site:linkedin.com "example corp" "senior engineer"
site:greenhouse.io "example" "software engineer"
```

Look for:
- Technology stack mentioned ("experience with AWS Lambda", "Kubernetes")
- Security tools ("experience with Splunk, Snyk")
- Internal system names mentioned in requirements

---

## Paste Sites

Developers paste things on Pastebin, Hastebin, etc. Sometimes accidentally sensitive.

```bash
# pwnbin — search pastes
pwnbin -k "example.com api_key"

# pastos
python3 pastos.py -s "example.com"
```

---

## Putting It Together

OSINT is about connecting dots. A job posting says they use Elasticsearch. Shodan shows port 9200 open on one of their IPs. That's a potential unauthenticated Elasticsearch instance — worth checking.

GitHub shows an old config file with a database hostname `db01.internal.example.com`. Subdomain brute force that namespace. Maybe it's accessible.

---

*Back to [[Recon-Overview]] | Related: [[Subdomain-Enumeration]] | [[JS-Recon]]*

*Sources: OWASP Testing Guide v4, HowToHunt, Bug Bounty Bootcamp*
