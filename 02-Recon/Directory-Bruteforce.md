# Directory & Endpoint Bruteforce

← [[Recon-Overview]]

---

Most web apps have endpoints that aren't linked from the UI — admin panels, debug routes, backup files, old API versions. Bruteforcing finds them by guessing common names from a wordlist.

---

## The Core Idea

Send `GET /FUZZ HTTP/1.1` with thousands of words substituted in for FUZZ. Look for responses that aren't 404.

---

## ffuf — The Go-To Tool

**ffuf** (Fuzz Faster U Fool) is the standard for web content discovery.

### Basic Directory Scan

```bash
ffuf -u https://example.com/FUZZ \
  -w /path/to/wordlist.txt \
  -mc 200,201,301,302,401,403 \
  -t 50
```

- `-mc` — match these status codes (exclude 404, 500)
- `-t 50` — 50 threads (be careful with rate limits)

### Filter Responses

```bash
# Filter by response size (exclude all 404s that return 1234 bytes)
ffuf -u https://example.com/FUZZ -w wordlist.txt -fs 1234

# Filter by word count
ffuf -u https://example.com/FUZZ -w wordlist.txt -fw 10

# Match only specific sizes (find unique responses)
ffuf -u https://example.com/FUZZ -w wordlist.txt -ms 0-500
```

### Scanning with Extensions

```bash
# Append extensions to each word
ffuf -u https://example.com/FUZZ \
  -w wordlist.txt \
  -e .php,.html,.js,.json,.bak,.old,.txt,.xml,.config \
  -mc 200,301,401,403
```

### API Endpoint Discovery

```bash
ffuf -u https://api.example.com/api/v1/FUZZ \
  -w /path/to/api-wordlist.txt \
  -mc 200,201,400,401,403 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### POST Parameter Fuzzing

```bash
ffuf -u https://example.com/login \
  -X POST \
  -d "username=FUZZ&password=test" \
  -w usernames.txt \
  -mc 200 \
  -H "Content-Type: application/x-www-form-urlencoded"
```

### Output to File

```bash
ffuf -u https://example.com/FUZZ \
  -w wordlist.txt \
  -mc 200,301,302,401,403 \
  -o results.json \
  -of json
```

---

## Wordlists

The quality of your wordlist determines your results. Store them in `~/wordlists/`.

### Best Wordlists from SecLists

```
SecLists/Discovery/Web-Content/
├── directory-list-2.3-medium.txt     ← 220k entries, general purpose
├── directory-list-2.3-small.txt      ← 87k entries, faster
├── raft-large-directories.txt        ← 62k unique dirs, high quality
├── raft-medium-files.txt             ← files with extensions
├── common.txt                        ← small, fast, good for quick checks
└── api/
    ├── api-endpoints.txt             ← API path wordlist
    └── objects.txt                   ← REST resource names
```

### Recommended Stack

1. Start with `raft-large-directories.txt` — good signal-to-noise ratio
2. Follow up with extensions on promising paths: `.php`, `.bak`, `.old`, `.gz`
3. Target-specific wordlist: add words from the app's own content (product names, features, technologies)

---

## gobuster

Alternative to ffuf — simpler syntax, slightly less flexible.

```bash
gobuster dir \
  -u https://example.com \
  -w /path/to/wordlist.txt \
  -x php,html,js,txt \
  -s 200,301,302,401,403 \
  -t 50
```

---

## Recursion

When you find a directory, scan inside it too.

```bash
# ffuf recursive
ffuf -u https://example.com/FUZZ \
  -w wordlist.txt \
  -mc 200,301,302,401,403 \
  -recursion \
  -recursion-depth 2
```

> [!WARNING]
> Recursive scanning with a large wordlist on many subdomains = huge request volume. Be mindful of rate limits and target server load, especially on bug bounty programs that prohibit DoS.

---

## Crawling First, Then Bruteforce

Before brute forcing, crawl the app to find real endpoints. Bruteforce only fills the gaps.

```bash
# katana — fast crawler
katana -u https://example.com -d 5 -jc -o crawl_results.txt

# Extract just paths
cat crawl_results.txt | grep "^https://example.com" | sed 's|https://example.com||' | sort -u > paths.txt
```

---

## What to Look For in Results

| Finding | Why interesting |
|---------|----------------|
| `/admin`, `/administrator` | Admin panel |
| `/backup`, `/backup.zip`, `/db.sql` | Exposed backups |
| `/.git/` | Exposed git repo — can download source code |
| `/.env` | Environment file — API keys, DB credentials |
| `/config.php`, `/config.json` | Config files |
| `/api/v0/`, `/api/v2/` | Older/newer API versions, potentially less hardened |
| `401` and `403` responses | Auth-protected — revisit after getting credentials |
| `/phpinfo.php` | PHP configuration dump — major info leak |
| `/server-status` | Apache server status page if misconfigured |

> [!TIP]
> `/.git/` exposure is critical. If you can access `/.git/config` or `/.git/HEAD`, use [git-dumper](https://github.com/arthaud/git-dumper) to download the entire repo source code.
>
> ```bash
> git-dumper https://example.com/.git/ ./dumped_repo
> ```

---

## 403 Bypass Techniques

Got a 403? Don't move on. It means something is there.

```bash
# Path variations
/admin         → /ADMIN, /Admin, /admin/, /admin/.
/admin         → /admin%20, /admin%09, /admin..;/, /admin;/

# Header overrides
curl -H "X-Original-URL: /admin" https://example.com/
curl -H "X-Rewrite-URL: /admin" https://example.com/
curl -H "X-Forwarded-For: 127.0.0.1" https://example.com/admin

# Method change
curl -X POST https://example.com/admin
curl -X HEAD https://example.com/admin
```

---

*Back to [[Recon-Overview]] | Related: [[Fingerprinting]] | [[JS-Recon]]*

*Sources: HowToHunt, ffuf documentation, SecLists*
