# How the Web Works

**Overview** | [[HTTP-Deep-Dive]] | [[Web-Architecture]] | [[Browser-Security-Model]]

---

Before you can hack a web application, you need to understand how it works. Every attack is just exploiting a misunderstanding or misconfiguration in this process.

---

## The Big Picture

When you type `https://example.com` in your browser and hit Enter, here's what actually happens:

```
You (Browser)  →  DNS  →  Server
     ↑                        |
     └──────── Response ───────┘
```

1. **DNS Lookup** — "What IP address is `example.com`?"
2. **TCP Connection** — a reliable connection is established with the server
3. **TLS Handshake** — encryption is negotiated (for HTTPS)
4. **HTTP Request** — your browser asks for a resource
5. **HTTP Response** — the server sends back content
6. **Rendering** — the browser turns HTML/CSS/JS into a visible page

Each one of these steps is an **attack surface**. Security bugs exist at every layer.

---

## Step 1 — DNS

DNS (Domain Name System) translates human-readable names to IP addresses.

```
example.com  →  DNS resolver  →  93.184.216.34
```

**Why it matters for security:**
- **DNS Spoofing / Cache Poisoning** — attacker feeds you a fake IP
- **Subdomain Takeover** — a subdomain points to a resource that no longer exists; attacker claims it
- **DNS Enumeration** — enumerating subdomains is one of the first recon steps

> [!NOTE]
> DNS responses are not encrypted by default. DNS over HTTPS (DoH) and DNS over TLS (DoT) fix this, but most corporate environments still use plain UDP port 53.

---

## Step 2 — TCP

TCP (Transmission Control Protocol) is how reliable connections are made.

The **three-way handshake**:
```
Client  →  SYN          →  Server
Client  ←  SYN-ACK      ←  Server
Client  →  ACK           →  Server
(connection established)
```

**Why it matters for security:**
- Port scanning (nmap) works by probing which ports respond to SYN
- Every open port is a potential entry point

---

## Step 3 — TLS (HTTPS)

TLS (Transport Layer Security) wraps HTTP in encryption so no one in the middle can read your traffic.

Key things TLS provides:
- **Confidentiality** — data is encrypted in transit
- **Integrity** — data can't be tampered with
- **Authentication** — you're talking to the real server (via certificates)

> [!WARNING]
> HTTPS only protects data **in transit**. If the app stores your password in plaintext in its database, HTTPS doesn't help once it hits the server.

**Why it matters for security:**
- Expired or misconfigured TLS certificates are common findings
- Tools like Burp Suite act as a **man-in-the-middle proxy** — they intercept your HTTPS traffic by installing a trusted CA cert in your browser
- This is exactly how attackers on an unprotected network can MITM you

---

## Step 4 — The HTTP Request

This is the core of web security. [[HTTP-Deep-Dive]] covers this in detail, but here's the shape of a request:

```http
GET /login HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Cookie: session=abc123
```

- **Method** — what action you want (`GET`, `POST`, `PUT`, `DELETE`, etc.)
- **Path** — which resource you want (`/login`)
- **Headers** — metadata about the request
- **Body** — data sent with `POST`/`PUT` requests (login forms, file uploads, JSON APIs)

Everything in a request is **attacker-controlled**. Every field is a potential injection point.

---

## Step 5 — The HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: session=xyz789; HttpOnly; Secure

<html>...</html>
```

- **Status code** — was the request successful?
- **Headers** — instructions to the browser (caching, cookies, security policies)
- **Body** — the actual content (HTML, JSON, images, etc.)

Security headers in responses (like `Content-Security-Policy`, `X-Frame-Options`) are a major topic in [[Browser-Security-Model]].

---

## Step 6 — The Browser Renders the Page

The browser parses HTML, fetches linked CSS/JS/images (more HTTP requests), runs JavaScript, and builds the DOM (Document Object Model).

JavaScript runs in a **sandboxed environment** — it can't access your filesystem, but it CAN:
- Make HTTP requests to other servers
- Read and modify cookies (unless `HttpOnly`)
- Read the DOM content of the current page

> [!NOTE]
> XSS (Cross-Site Scripting) exploits this — if an attacker can inject JavaScript into a page, it runs in the victim's browser with full access to that page's context.

---

## Summary

| Layer | Protocol | Key Security Concerns |
|-------|----------|----------------------|
| Name resolution | DNS | Spoofing, subdomain takeover |
| Transport | TCP | Port scanning, firewall bypass |
| Encryption | TLS | MITM, cert misconfig, downgrade |
| Application | HTTP | Every vuln class you'll ever hunt |
| Client-side | Browser/JS | XSS, CORS, clickjacking |

---

## What's Next

Now that you know the request/response cycle, dive into the details:

- [[HTTP-Deep-Dive]] — methods, headers, cookies, status codes in depth
- [[Web-Architecture]] — understanding multi-tier apps, APIs, CDNs, and what you're actually targeting
- [[Browser-Security-Model]] — why browsers have the Same-Origin Policy and what happens when it breaks down

---

*Sources: MDN Web Docs, RFC 2616/7230, OWASP Testing Guide v4*
