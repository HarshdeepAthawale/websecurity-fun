# Browser Security Model

[[How-the-Web-Works]] | [[HTTP-Deep-Dive]] | [[Web-Architecture]] | **Overview** | [[Same-Origin-Policy]] | [[CORS]] | [[CSP]]

---

The browser is the most powerful client-side security system on the web. It also has to balance security against the need for the web to actually work. That tension is where many vulnerabilities come from.

---

## Why the Browser Has to Enforce Security

JavaScript running in your browser is powerful. It can:
- Make HTTP requests to any server (fetch, XMLHttpRequest)
- Read and modify the DOM
- Access cookies (unless `HttpOnly`)
- Read localStorage and sessionStorage
- Open new windows/tabs
- Communicate with other tabs from the same origin

Without restrictions, visiting `evil.com` could have that page's JS:
- Read your bank's page if you had it open
- Send requests to your bank's API using your session cookies
- Steal all your local data

The browser's security model prevents this.

---

## The Same-Origin Policy (SOP)

The **Same-Origin Policy** is the foundational security rule of the web.

**Origin = Scheme + Host + Port**

| URL | Origin |
|-----|--------|
| `https://example.com/page` | `https://example.com` |
| `https://example.com:443/other` | `https://example.com` (443 is default for https) |
| `http://example.com/page` | `http://example.com` (different scheme!) |
| `https://sub.example.com/page` | `https://sub.example.com` (different host!) |
| `https://example.com:8080/page` | `https://example.com:8080` (different port!) |

**The rule:** JavaScript can only freely read responses from the **same origin**. Requests to other origins are restricted.

> [!NOTE]
> SOP restricts **reading** responses. It doesn't prevent **sending** requests. This is why CSRF works — the browser will send a cross-origin request with your cookies, it just won't let the attacking page *read* the response.

See [[Same-Origin-Policy]] for the full deep dive including what IS and ISN'T covered by SOP.

---

## CORS — Relaxing SOP for APIs

Modern web apps need to make legitimate cross-origin requests. A React SPA on `app.example.com` needs to call an API on `api.example.com`.

CORS (Cross-Origin Resource Sharing) is the mechanism that lets servers selectively allow cross-origin requests.

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

**Misconfigured CORS is a major vulnerability.** Common mistakes:
- Reflecting the `Origin` header back without validation
- Using `Access-Control-Allow-Origin: *` with `Allow-Credentials: true` (browsers block this, but logic is still wrong)
- Trusting origins like `null` or via overly broad regex

See [[CORS]] for the full attack surface.

---

## Content Security Policy (CSP)

CSP is a response header that tells the browser where it's allowed to load resources from. It's the main defense against XSS.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

This says: "Only run JavaScript from this domain and cdn.example.com. Block everything else."

**Why it matters:**
- A good CSP can **neutralize** an XSS vuln by preventing injected scripts from running
- A weak or missing CSP means XSS is fully exploitable
- CSP bypasses are their own research area — `unsafe-inline`, `unsafe-eval`, CDN bypasses, JSONP endpoints

See [[CSP]] for a full breakdown of directives and bypass techniques.

---

## Cookies and the Browser

The browser manages cookies and sends them automatically with every request to the matching domain/path. Three flags matter most:

| Flag | Effect |
|------|--------|
| `HttpOnly` | JS cannot read this cookie. XSS can't steal it directly. |
| `Secure` | Only sent over HTTPS. |
| `SameSite=Strict` | Not sent on cross-site requests at all. Prevents CSRF. |
| `SameSite=Lax` | Sent on top-level navigations (link clicks). Partial CSRF protection. |
| `SameSite=None` | Always sent cross-site (requires `Secure`). |

> [!WARNING]
> `HttpOnly` protects against **cookie theft** via XSS, but XSS can still *use* the cookie by making requests from the victim's browser — the JS doesn't need to read the cookie value.

---

## The Browser's Trust Hierarchy

```
HTTPS + Valid Cert = Encrypted + Authentic
    ↓ but...
Content from the origin can still be malicious (XSS)
Content from third-party scripts (ads, analytics) runs with same privilege
Extensions run with elevated privilege
```

**Key insight:** The browser trusts that content from an origin is legitimate. If an attacker can inject content into a legitimate origin (XSS), the browser has no way to distinguish it from real content.

---

## Clickjacking

The browser allows pages to be embedded in `<iframe>` tags. An attacker can layer a transparent iframe over a fake page, tricking users into clicking on elements of the real site.

Defense: `X-Frame-Options: DENY` or `Content-Security-Policy: frame-ancestors 'none'`

---

## Cross-Site Request Forgery (CSRF) — How the Browser Enables It

When you visit `evil.com`, any JavaScript there can trigger a request to `bank.com`:

```html
<form action="https://bank.com/transfer" method="POST">
  <input name="amount" value="1000">
  <input name="to" value="attacker">
</form>
<script>document.forms[0].submit()</script>
```

The browser **automatically includes your `bank.com` cookies** in this request. The bank sees a valid session cookie. It processes the transfer.

Defenses: CSRF tokens, `SameSite` cookies, checking the `Origin`/`Referer` header.

This only works because the **attacker doesn't need to read the response** — they just need the action to happen.

---

## Browser Developer Tools for Security Testing

Before Burp Suite, understand your browser's built-in tools:

| Tool | How to use it |
|------|--------------|
| **Network tab** | See every HTTP request/response. Right-click → Copy as cURL |
| **Console** | Run JS, see errors and logs |
| **Application tab** | View/edit cookies, localStorage, sessionStorage |
| **Sources tab** | Read all loaded JS files, set breakpoints |
| **Elements tab** | Inspect and modify the DOM |

> [!TIP]
> `Right-click any request in the Network tab → Copy → Copy as fetch` gives you JavaScript you can paste in the console to replay that request. Great for quick testing without Burp.

---

## Summary

| Mechanism | Protects Against | Can Be Bypassed By |
|-----------|-----------------|-------------------|
| Same-Origin Policy | Cross-origin JS reads | CORS misconfig, XSS |
| CORS | Unintended cross-origin access | Misconfigured `Allow-Origin` |
| CSP | XSS exploitation | Weak policy, JSONP endpoints |
| `HttpOnly` | Cookie theft via XSS | Still vulnerable to requests |
| `SameSite` | CSRF | `Lax` still allows some cases |
| `X-Frame-Options` | Clickjacking | CSP `frame-ancestors` is better |

---

## What's Next

- [[Same-Origin-Policy]] — the details of what SOP does and doesn't protect
- [[CORS]] — attack surface of cross-origin resource sharing
- [[CSP]] — directives, bypass techniques
- [[02-Recon/Recon-Overview]] — now that you understand the web, start mapping targets

---

*Sources: MDN Web Docs, PortSwigger Web Security Academy, OWASP Cheat Sheet Series*
