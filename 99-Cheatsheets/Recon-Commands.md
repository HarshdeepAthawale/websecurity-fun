# Recon Commands Cheatsheet

---

## Subdomain Enumeration

```bash
# Passive
subfinder -d TARGET.com -all -silent -o subs.txt
curl -s "https://crt.sh/?q=%.TARGET.com&output=json" | jq -r '.[].name_value' | sort -u

# Resolve + live check
cat subs.txt | dnsx -silent | httpx -silent -status-code -title -tech-detect

# Active brute force
dnsx -d TARGET.com -w ~/wordlists/subdomains-top1million.txt -silent

# Subdomain takeover
subjack -w subs.txt -t 100 -timeout 30 -ssl -o takeovers.txt
nuclei -l subs.txt -t ~/nuclei-templates/takeovers/
```

---

## Content Discovery

```bash
# Directory brute force
ffuf -u https://TARGET.com/FUZZ -w ~/wordlists/raft-large-directories.txt -mc 200,301,302,401,403 -t 50

# With extensions
ffuf -u https://TARGET.com/FUZZ -w ~/wordlists/raft-large-files.txt -e .php,.bak,.old,.txt,.env,.json -mc 200,301,302,403

# API endpoints
ffuf -u https://api.TARGET.com/api/v1/FUZZ -w ~/wordlists/api-endpoints.txt -mc 200,201,400,401,403

# Parameter discovery
arjun -u https://TARGET.com/endpoint -m GET
arjun -u https://TARGET.com/endpoint -m POST
```

---

## URL Collection

```bash
# Historical URLs
waybackurls TARGET.com | sort -u > wayback.txt
gau --threads 5 TARGET.com | sort -u > gau.txt

# Combine and filter
cat wayback.txt gau.txt | sort -u | grep -v "\.(png|jpg|gif|css|woff|svg)" > urls.txt

# Interesting files
cat urls.txt | grep -iE "\.(php|aspx|js|json|xml|yaml|env|bak|sql|zip|tar|gz)$"
```

---

## JS Recon

```bash
# Crawl and find JS files
katana -u https://TARGET.com -d 5 -jc | grep "\.js$" | sort -u > js_files.txt

# Extract endpoints from JS
cat js_files.txt | xargs -I{} python3 linkfinder.py -i {} -o cli 2>/dev/null | sort -u

# Hunt for secrets
cat js_files.txt | xargs -I{} curl -sk {} | grep -oP 'AKIA[0-9A-Z]{16}'  # AWS keys
cat js_files.txt | xargs -I{} curl -sk {} | grep -iE "(api_key|apikey|secret|password|token)" | head -50
```

---

## Fingerprinting

```bash
# Tech detect
httpx -u https://TARGET.com -tech-detect -title -status-code

# WAF detection
wafw00f https://TARGET.com

# Full headers
curl -sI https://TARGET.com

# WhatWeb
whatweb -a 3 https://TARGET.com
```

---

## Google Dorks (replace TARGET.com)

```
site:TARGET.com ext:env OR ext:log OR ext:sql OR ext:bak
site:TARGET.com inurl:admin OR inurl:login OR inurl:dashboard
site:TARGET.com intitle:"Index of /"
site:*.TARGET.com -www
site:TARGET.com inurl:swagger OR inurl:api-docs OR inurl:graphql
"TARGET.com" site:github.com
```

---

## Shodan

```bash
# CLI
shodan search --fields ip_str,port,org "org:TARGET"
shodan host TARGET_IP

# Web queries
hostname:TARGET.com port:8080
hostname:TARGET.com port:27017
http.title:"Admin" org:"Target Corp"
```

---

## Full Pipeline (quick start)

```bash
TARGET="example.com"
mkdir -p recon/$TARGET && cd recon/$TARGET

# 1. Subdomains
subfinder -d $TARGET -all -silent -o subs_passive.txt
cat subs_passive.txt | dnsx -silent -o subs_resolved.txt
cat subs_resolved.txt | httpx -silent -status-code -title -tech-detect -o live_hosts.txt

# 2. URLs
gau --threads 5 $TARGET | sort -u > urls.txt
waybackurls $TARGET | sort -u >> urls.txt
sort -u urls.txt -o urls.txt

# 3. Quick content discovery on each live host
cat live_hosts.txt | awk '{print $1}' | while read host; do
  ffuf -u "$host/FUZZ" -w ~/wordlists/common.txt -mc 200,301,302,401,403 -t 30 -o "ffuf_$host.json" -of json -s
done
```
