# Subdomain Enumeration

← [[Recon-Overview]]

---

Subdomains are the most overlooked attack surface. The main domain is hardened. The `dev.`, `legacy.`, `internal.` subdomain that someone forgot about is not.

---

## Why Subdomains Matter

- Different subdomains often run different apps, different tech stacks, different teams
- Dev/staging environments have weaker security controls
- Old subdomains may point to decommissioned infrastructure (→ subdomain takeover)
- Internal tools accidentally exposed: `jira.example.com`, `gitlab.example.com`

---

## Passive Enumeration (No Direct Contact with Target)

Passive means you're querying third-party data sources — the target never sees your requests.

### Certificate Transparency (crt.sh)

Every SSL cert issued is logged publicly. This is the most reliable passive source.

```bash
curl -s "https://crt.sh/?q=%.example.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u > ct_subs.txt
```

### Subfinder

Aggregates multiple passive sources (crt.sh, VirusTotal, SecurityTrails, etc.) in one tool.

```bash
subfinder -d example.com -all -silent -o subfinder_subs.txt
```

### Chaos (ProjectDiscovery)

Free dataset of subdomains for thousands of programs updated regularly.

```bash
# First time: download the dataset for your target
chaos -d example.com -o chaos_subs.txt
```

### SecurityTrails / Shodan / VirusTotal

Commercial intel sources with historical subdomain data. Useful for finding subdomains that no longer resolve but used to exist (old infrastructure).

### Aggregating Passive Results

```bash
cat ct_subs.txt subfinder_subs.txt chaos_subs.txt | sort -u > all_passive_subs.txt
echo "Total unique: $(wc -l < all_passive_subs.txt)"
```

---

## Active Enumeration (DNS Brute Force)

You're querying DNS resolvers directly with a wordlist. The target's DNS server handles your requests — this is "louder" but finds subdomains that never got certs.

### dnsx

Fast DNS resolver and brute-forcer.

```bash
# Resolve passive results first
cat all_passive_subs.txt | dnsx -silent -o resolved_subs.txt

# Brute force with wordlist
dnsx -d example.com -w subdomains-top1million.txt -silent -o brute_subs.txt
```

### ffuf (DNS mode)

```bash
ffuf -u https://FUZZ.example.com -w subdomains-top1million.txt \
  -mc 200,301,302,401,403 -t 50 -o ffuf_subs.json
```

### Good Wordlists

- `SecLists/Discovery/DNS/subdomains-top1million-110000.txt` — 1M most common subdomains
- `all.txt` from Jason Haddix — very large, thorough
- Custom wordlist from target-specific words (product names, team names from LinkedIn)

---

## Live Host Discovery

A subdomain resolving in DNS doesn't mean there's a web server behind it. Filter to actual live hosts:

```bash
# httpx probes for HTTP/HTTPS and returns status codes, tech, titles
cat resolved_subs.txt | httpx -silent -status-code -title -tech-detect -o live_hosts.txt

# Example output:
# https://admin.example.com [200] [Admin Panel] [nginx,PHP]
# https://dev.example.com [401] [Dev Environment] [Apache]
# https://legacy.example.com [403] [] [IIS]
```

---

## Subdomain Takeover

A subdomain pointing to an unclaimed resource is a takeover opportunity.

**How it happens:**
1. `staging.example.com` was set up with a CNAME to `example-staging.herokuapp.com`
2. The Heroku app was deleted
3. The DNS record was never cleaned up
4. The CNAME is dangling — anyone can claim `example-staging.herokuapp.com`

**Finding dangling CNAMEs:**

```bash
# subjack checks for takeover vulnerabilities
subjack -w all_subs.txt -t 100 -timeout 30 -o takeovers.txt -ssl

# nuclei has subdomain takeover templates
nuclei -l all_subs.txt -t nuclei-templates/takeovers/ -o takeover_results.txt
```

**Common takeover services:**
- Heroku, GitHub Pages, Netlify, Vercel, AWS S3, Azure, Fastly, Ghost, Zendesk

> [!NOTE]
> Subdomain takeover severity depends on the subdomain. `logout.example.com` takeover = low. `auth.example.com` takeover = critical (cookie scope, auth flows).

---

## Permutation / Alterations

Generate variations of known subdomains to find more:

```bash
# gotator generates permutations
gotator -sub resolved_subs.txt -perm permutations.txt -depth 1 -silent | \
  dnsx -silent -o permutation_results.txt
```

Examples of permutations:
- `api.example.com` → `api-dev.example.com`, `api-v2.example.com`, `api-staging.example.com`
- `admin.example.com` → `admin2.example.com`, `admin-portal.example.com`

---

## Full Pipeline

```bash
TARGET="example.com"

# 1. Passive
subfinder -d $TARGET -all -silent -o subs_passive.txt
curl -s "https://crt.sh/?q=%.${TARGET}&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u >> subs_passive.txt
sort -u subs_passive.txt -o subs_passive.txt

# 2. Resolve
cat subs_passive.txt | dnsx -silent -o subs_resolved.txt

# 3. Active brute force
dnsx -d $TARGET -w ~/wordlists/subdomains-top1million.txt -silent >> subs_resolved.txt
sort -u subs_resolved.txt -o subs_resolved.txt

# 4. Live hosts
cat subs_resolved.txt | httpx -silent -status-code -title -tech-detect -o live_hosts.txt

# 5. Check takeovers
subjack -w subs_resolved.txt -t 100 -timeout 30 -o takeovers.txt -ssl
```

---

*Back to [[Recon-Overview]] | Related: [[Fingerprinting]] | [[OSINT]]*

*Sources: HowToHunt, ProjectDiscovery docs, Bug Bounty Bootcamp*
