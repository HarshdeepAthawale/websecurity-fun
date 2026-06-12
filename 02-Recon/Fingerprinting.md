# Fingerprinting

← [[Recon-Overview]]

---

Fingerprinting is identifying what technology a target is running. It sounds boring but it's strategic — different tech stacks have different vulnerability patterns, different default misconfigs, and different known CVEs.

---

## Why It Matters

- **PHP** → look for LFI, file upload vulns, old CVEs in CMS (WordPress, Joomla)
- **Java** → look for deserialization, XXE, Spring-specific vulns
- **Node.js/Express** → prototype pollution, SSTI in template engines
- **AWS infrastructure** → SSRF to metadata endpoint, S3 misconfig
- **nginx** → alias traversal, misconfigured proxy rules
- **Old CMS versions** → known exploits, unauthenticated endpoints

---

## Fingerprinting from HTTP Headers

The easiest fingerprinting — just look at response headers.

```bash
curl -sI https://example.com
```

| Header | What it reveals |
|--------|----------------|
| `Server: Apache/2.4.49` | Web server + version (CVE lookup time) |
| `Server: nginx/1.18.0` | Nginx version |
| `X-Powered-By: PHP/7.4.3` | PHP version |
| `X-Powered-By: Express` | Node.js + Express |
| `X-AspNet-Version: 4.0.30319` | .NET version |
| `X-Generator: Drupal 9` | CMS |
| `Set-Cookie: PHPSESSID=...` | PHP backend |
| `Set-Cookie: JSESSIONID=...` | Java backend |
| `Set-Cookie: ASP.NET_SessionId=...` | ASP.NET |
| `Set-Cookie: laravel_session=...` | Laravel (PHP) |
| `Set-Cookie: _rails_session=...` | Ruby on Rails |
| `CF-Ray: ...` | Behind Cloudflare |
| `X-Cache: Hit from cloudfront` | AWS CloudFront CDN |

---

## Fingerprinting from URLs and Paths

| Pattern | Technology |
|---------|-----------|
| `/wp-content/`, `/wp-admin/` | WordPress |
| `/joomla/`, `/administrator/` | Joomla |
| `/drupal/`, `?q=node/` | Drupal |
| `.php` extension | PHP |
| `.aspx`, `.asmx` | ASP.NET |
| `.jsp`, `.do`, `.action` | Java (Struts, Spring MVC) |
| `/_next/`, `/static/chunks/` | Next.js |
| `/api/v1/`, `/graphql` | API structure hints |
| `/__webpack_hmr` | React/webpack dev server |

---

## Fingerprinting from HTML and JS

Look at the page source:

```bash
curl -s https://example.com | grep -iE "(generator|framework|powered|built with)"
```

Common markers:
- `<meta name="generator" content="WordPress 6.2">` — version in meta tag
- Angular: `ng-version` attribute in `<app-root>` tag
- React: `__REACT_FIBER_TYPE__` in JS, `data-reactroot` in HTML
- Vue: `__vue_app__` in JS globals
- jQuery version: `jquery-3.6.0.min.js` in source

---

## Fingerprinting WAFs

Knowing the WAF helps you choose bypass techniques.

```bash
wafw00f https://example.com
```

Manual tells:
- Cloudflare: `CF-Ray` header, `__cfduid` cookie, blocks return `403` with Cloudflare branding
- Akamai: `AkamaiGHost` in Server header
- AWS WAF: `x-amz-cf-id` header
- Imperva: `incap_ses_` cookie, `visid_incap_` cookie
- F5 BIG-IP: `BIGipServer` cookie

---

## Fingerprinting with Tools

### httpx

```bash
cat live_hosts.txt | httpx -tech-detect -title -status-code -silent
```

httpx uses Wappalyzer signatures to detect:
- Frameworks (React, Angular, Vue, Django, Rails, Laravel)
- Servers (nginx, Apache, IIS)
- CDNs (Cloudflare, Fastly, Akamai)
- Analytics and tracking tools
- CMS platforms

### WhatWeb

```bash
whatweb -a 3 https://example.com
```

### Wappalyzer

Browser extension — shows detected technologies in the toolbar as you browse. Fastest for manual testing.

---

## CMS-Specific Fingerprinting

### WordPress

```bash
# WPScan — specialized WordPress scanner
wpscan --url https://example.com --enumerate u,p,t

# Manual: version in readme or meta
curl -s https://example.com/readme.html | grep "Version"
curl -s https://example.com | grep "wp-content"
```

Key WordPress attack surface:
- `/wp-login.php` — brute force target
- `/xmlrpc.php` — legacy API, often vulnerable
- `/wp-json/wp/v2/users` — user enumeration if not disabled
- Outdated plugins — #1 source of WordPress vulns

### Drupal

```bash
# Droopescan
droopescan scan drupal -u https://example.com
```

---

## Building a Fingerprint Profile

For each host in scope, record:

```markdown
## staging.example.com

- **Server:** nginx/1.18.0
- **Backend:** PHP/7.4 (from X-Powered-By)
- **Framework:** Laravel (laravel_session cookie)
- **CDN/WAF:** Cloudflare
- **CMS:** None
- **Interesting:** /admin returns 302 → /admin/login, /api/v1 returns JSON
- **Notes:** Older PHP version — check for specific CVEs
```

---

*Back to [[Recon-Overview]] | Related: [[Subdomain-Enumeration]] | [[Directory-Bruteforce]]*

*Sources: OWASP Testing Guide v4, HowToHunt*
