# JavaScript Recon

← [[Recon-Overview]]

---

Modern web apps are mostly JavaScript. Developers build React/Vue/Angular frontends that contain a wealth of information: API endpoints, auth logic, internal paths, hardcoded tokens, feature flags, comments. This stuff is sitting in the browser — most people just don't look at it.

---

## Why JS Files Are Gold

JS bundles contain:
- API endpoint paths the UI doesn't expose yet
- Internal service URLs (`https://internal-api.corp.example.com`)
- Hardcoded API keys, AWS keys, tokens
- Auth logic that can be reverse-engineered
- Route configurations exposing all app paths
- Debug endpoints left in production

---

## Step 1 — Find All JS Files

### From Browser

Open DevTools → Sources tab → look for `.js` files. Pay attention to large bundle files.

### From Crawling

```bash
# katana crawls and extracts all JS URLs
katana -u https://example.com -d 5 -jc -o crawl.txt
grep "\.js" crawl.txt | sort -u > js_files.txt
```

### Wayback Machine

Old JS files in Wayback can contain endpoints/secrets that were removed:

```bash
waybackurls example.com | grep "\.js$" | sort -u > wayback_js.txt
```

### gau (GetAllURLs)

```bash
gau example.com | grep "\.js$" | sort -u >> js_files.txt
```

---

## Step 2 — Download and Analyze

```bash
# Download all JS files
cat js_files.txt | xargs -I{} curl -s {} -o {##}.js
```

Or use `hakrawler` / `gospider` which download automatically.

---

## Step 3 — Extract Endpoints

### Manual grep

```bash
# URLs and paths in JS
grep -oP '(https?://[^"'"'"'\s]+|/[a-zA-Z0-9_/-]+\.[a-zA-Z]{2,4})' *.js | sort -u

# API paths specifically
grep -oP '/api/[a-zA-Z0-9/_-]+' *.js | sort -u
```

### LinkFinder

Specifically designed to extract endpoints from JS files.

```bash
python3 linkfinder.py -i https://example.com/static/app.js -o cli
```

### secretfinder

```bash
python3 SecretFinder.py -i https://example.com/app.js -o cli
```

---

## Step 4 — Hunt for Secrets

### Manual Patterns

```bash
# API keys (generic high-entropy strings)
grep -oP '["\x27][A-Za-z0-9_\-]{20,40}["\x27]' *.js

# AWS keys
grep -oP 'AKIA[0-9A-Z]{16}' *.js

# JWT tokens
grep -oP 'eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+' *.js

# Common patterns
grep -iE "(api_key|apikey|api-key|access_token|secret|password|passwd|token|auth)" *.js
```

### truffleHog / gitleaks

For scanning many files at once:

```bash
trufflehog filesystem ./js_files/ --only-verified

gitleaks detect --source ./js_files/ -v
```

---

## Step 5 — Deobfuscate / Prettify

Production bundles are minified (everything on one line, variable names are `a`, `b`, `c`). Prettify them first:

```bash
# js-beautify
js-beautify -o app_pretty.js app.min.js
```

Or paste into an online prettifier (careful with sensitive targets — don't paste confidential code online).

For webpack bundles specifically, look for:
- Module IDs and their paths — reveals the app's internal structure
- Comment strings left in minified code
- Source map files (`.js.map`) — contain the **original, unminified source code**

> [!TIP]
> Check for `https://example.com/static/js/main.js.map` — if the source map file is exposed, you can see the original developer source code, complete with variable names, comments, and file structure. This is a critical information disclosure finding.

---

## Step 6 — Extract Routes (SPA Apps)

React Router, Vue Router, Angular Router — they all define every route in the JS bundle.

```bash
# React Router routes
grep -oP "path:\s*['\"][^'\"]+['\"]" *.js | sort -u

# Generic route patterns
grep -oP '"route":\s*"[^"]+"' *.js | sort -u
grep -oP "path\s*=\s*['\"][^'\"]+['\"]" *.js | sort -u
```

You're looking for routes like `/admin/*`, `/internal/*`, `/debug` that aren't linked in the normal UI.

---

## Real-World Findings From JS

- Hardcoded internal API URL leaking internal architecture
- AWS S3 bucket name in a JS config object → check if public
- Firebase config object with `apiKey` and `authDomain` → test for misconfigured Firebase rules
- Staging/dev API URL left in production bundle
- Admin feature code for functionality not shown in UI (but still active on backend)
- JWT secret used to sign tokens (in a debugging helper)

---

*Back to [[Recon-Overview]] | Related: [[Directory-Bruteforce]] | [[OSINT]]*

*Sources: HowToHunt, Bug Bounty Bootcamp, NahamSec recon methodology*
