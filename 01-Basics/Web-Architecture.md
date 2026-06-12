# Web Architecture

[[How-the-Web-Works]] | [[HTTP-Deep-Dive]] | **Overview** | [[Browser-Security-Model]]

---

Knowing the request/response cycle is step one. But real web applications have many moving parts — and every component is a potential target. This note maps out what you're actually attacking when you test a web app.

---

## The Simple Model (What Most Think It Is)

```
Browser  →  Server  →  Database
```

Most tutorials stop here. Reality is messier.

---

## The Real Model

```
                         ┌─────────────────────────────────────┐
                         │           The Internet               │
                         └─────────────────────────────────────┘
                                          │
                                    ┌─────▼──────┐
                                    │    CDN /    │
                                    │  WAF / LB   │  ← Layer 1: Edge
                                    └─────┬───────┘
                                          │
                                  ┌───────▼────────┐
                                  │   Web Server   │  ← Layer 2: Web/App
                                  │ (nginx/Apache) │
                                  └───────┬────────┘
                                          │
                               ┌──────────▼──────────┐
                               │   Application Layer  │  ← Layer 3: Business Logic
                               │ (Node, Django, Rails) │
                               └──────────┬───────────┘
                                          │
                    ┌─────────────────────▼──────────────────────┐
                    │                                              │
              ┌─────▼──────┐   ┌──────────┐   ┌──────────────┐  │
              │  Database   │   │  Cache   │   │  File Store  │  │
              │(SQL/NoSQL)  │   │  (Redis) │   │  (S3/disk)   │  │  ← Layer 4: Data
              └─────────────┘   └──────────┘   └──────────────┘  │
                                                                   └
```

Let's look at each layer from an attacker's perspective.

---

## Layer 1: Edge (CDN / WAF / Load Balancer)

### CDN (Content Delivery Network)

A CDN caches static content (images, JS, CSS) closer to users geographically. Services like Cloudflare, Akamai, and Fastly act as CDNs.

**Security relevance:**
- CDNs often also act as WAFs (Web Application Firewalls)
- **Cache poisoning** — if you can control what gets cached, you can serve malicious content to other users
- CDNs can hide the **real origin IP** — finding it is a recon goal (check DNS history, old records, certificates)
- **Subdomain takeover** often happens when a CDN/S3 mapping goes stale

### WAF (Web Application Firewall)

A WAF sits in front of the app and blocks malicious-looking requests (SQLi, XSS, etc.).

**Security relevance:**
- WAFs are an **obstacle**, not a solution — they can be bypassed
- WAF bypass techniques include encoding payloads, using alternative syntax, case variations, HTTP chunked encoding
- WAF detection: look for `403` or odd `400` responses to payloads, check response headers for `cf-ray` (Cloudflare), `X-CDN` etc.

> [!TIP]
> If a WAF blocks your SQLi payload but the raw app is vulnerable, your task is WAF bypass — not finding a new vuln. See [[WAF-Bypass-Techniques]].

### Load Balancer

Distributes traffic across multiple backend servers.

**Security relevance:**
- Can cause issues with session affinity — one request goes to Server A, next to Server B
- Useful in race condition exploits — sometimes different nodes handle the race differently

---

## Layer 2: Web Server

This is the software that receives your HTTP request and routes it.

Common web servers:
- **Nginx** — very common, used as a reverse proxy
- **Apache** — older, still very common
- **IIS** — Microsoft environments
- **Caddy**, **Lighttpd** — less common

**Security relevance:**
- Web server **misconfigurations** are a major finding category
- Nginx `alias` + `location` mismatch → **path traversal**
- Apache `.htaccess` misconfiguration → file exposure
- Default pages left up → fingerprinting
- Version in `Server` header → known CVE lookup

> [!NOTE]
> The web server often handles **static files** itself and only passes **dynamic requests** to the application layer. Understanding where this boundary is matters — a `/uploads/shell.php` might not execute if the web server is Nginx and it's not forwarding `.php` to PHP-FPM.

---

## Layer 3: Application Layer

This is where your actual business logic runs. The app code — in Node.js, Python/Django/Flask, Ruby on Rails, PHP, Java Spring, Go, etc.

**This is where most web vulnerabilities live.**

Key concepts:

### Routing
The app maps URL paths to functions/handlers. `/user/123/profile` might resolve to `userController.getProfile(id=123)`.

**Security relevance:** Can you access `/user/124/profile` without permission? → **IDOR**

### Authentication vs Authorization
- **Authentication** = who are you? (login, tokens, MFA)
- **Authorization** = what are you allowed to do? (roles, permissions)

These are two separate systems and both can be broken independently.

### Input Handling
Everything the user sends is **untrusted input**. If the app doesn't sanitize it:
- Into an SQL query → **SQLi**
- Into an HTML page → **XSS**
- Into a shell command → **Command Injection**
- Into a file path → **Path Traversal**
- Into an XML parser → **XXE**

---

## Layer 4: Data Layer

### Databases
- **Relational (SQL):** PostgreSQL, MySQL, MSSQL, SQLite — data in tables with relationships
- **NoSQL:** MongoDB, Redis, Cassandra — flexible schemas, document/key-value stores

**Security relevance:**
- SQL injection → [[SQL-Injection]]
- NoSQL injection → different syntax but same concept (`$where`, `$regex` operators)
- Overprivileged DB user — app uses a DB account with `DROP TABLE` permissions when it only needs `SELECT`

### Cache (Redis, Memcached)
Stores frequently-accessed data in memory for speed.

**Security relevance:**
- Unauthenticated Redis is a classic critical finding — direct RCE in some cases
- Cache poisoning — storing attacker-controlled data in the cache

### File Storage (S3, local disk)
Uploaded files, generated reports, profile pictures.

**Security relevance:**
- **Unrestricted File Upload** → [[File-Upload-Vulnerabilities]]
- **S3 bucket misconfiguration** → public bucket exposes sensitive files
- **Path traversal** in file download endpoints

---

## APIs and the Modern App

Modern apps are often split into:
- A **frontend** (React/Vue/Angular SPA) that runs in the browser
- A **backend API** (REST or GraphQL) that the frontend calls

**Why this matters for security:**
- The API is the real attack surface — not the HTML page
- APIs often have weaker auth checks (mobile clients, assumed trusted)
- **GraphQL** has its own vuln class — introspection leaks the full schema, batching enables brute force, etc. → [[GraphQL-Security]]
- API versioning: `/api/v1/users` might be hardened; `/api/v2/users` might not

---

## Server-Side vs Client-Side

| | Server-Side | Client-Side |
|---|---|---|
| Runs on | Server (you can't see it directly) | Browser (you can read the JS source) |
| Attacks | SQLi, SSRF, XXE, RCE, path traversal | XSS, CSRF, clickjacking, prototype pollution |
| Trust boundary | Input comes from untrusted users | Code runs in untrusted environment |

---

## What You're Mapping During Recon

When you start testing a target, you're building a mental map:
1. What's at the edge? (CDN? WAF? Which one?)
2. What web server is running?
3. What tech stack? (Headers, cookies, error pages, job postings all leak this)
4. What's the API structure? REST or GraphQL?
5. What data does it store? What's sensitive?
6. Where are the trust boundaries?

---

## What's Next

- [[Browser-Security-Model]] — how the browser enforces security on the client side
- [[02-Recon/Recon-Overview]] — how to map a target's architecture methodically
- [[03-OWASP-Top-10/OWASP-Overview]] — the vulnerability classes that affect every layer

---

*Sources: OWASP Testing Guide v4, PortSwigger Web Security Academy, MDN Web Docs*
