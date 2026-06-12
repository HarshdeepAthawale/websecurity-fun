# Same-Origin Policy (SOP)

← [[Browser-Security-Model]]

---

The Same-Origin Policy is the browser's core security rule. Everything else — CORS, CSRF, XSS impact — only makes sense once you understand SOP.

---

## What Is an "Origin"?

An origin is defined by three components:

```
scheme://host:port
```

| URL | Origin | Same as https://example.com? |
|-----|--------|------------------------------|
| `https://example.com/page` | `https://example.com` | ✅ Yes |
| `https://example.com:443/other` | `https://example.com` | ✅ Yes (443 is implicit) |
| `http://example.com/page` | `http://example.com` | ❌ No (different scheme) |
| `https://sub.example.com` | `https://sub.example.com` | ❌ No (different host) |
| `https://example.com:8080` | `https://example.com:8080` | ❌ No (different port) |

All three parts must match. One difference = different origin.

---

## The Rule

JavaScript running on `https://example.com` can:

✅ **Read** responses from `https://example.com` (same origin)
✅ **Send** requests to any origin (but can't read the response)
❌ **Read** responses from `https://other.com` (cross-origin, blocked by SOP)

---

## What SOP Does and Doesn't Protect

### What SOP Protects

- JS on `evil.com` can't read your `bank.com` page content
- JS on `evil.com` can't read `bank.com` API responses
- JS on `evil.com` can't access the DOM of a `bank.com` iframe (if origins differ)
- JS on `evil.com` can't read `bank.com` cookies (only the browser sends them)

### What SOP Does NOT Protect

- **Sending requests** — JS on `evil.com` CAN send a POST to `bank.com`. It just can't read the response. This is why CSRF is possible.
- **Simple resource loads** — `<img src="https://other.com/img.png">` loads fine. `<script src="...">` loads and executes JS from other origins (this is how CDNs work).
- **Form submissions** — HTML forms submit cross-origin by default. CSRF exploits this.
- **Redirects** — you can redirect a user to any URL

> [!WARNING]
> This is the most misunderstood part of SOP: **it restricts reading, not sending**. CSRF works because the browser sends the request (including cookies) but the attacking page can't read the response. That's often enough — transferring money doesn't require reading the response.

---

## SOP and iframes

If you embed `https://bank.com` in an iframe on `https://evil.com`:

- You can't read the iframe's DOM content (SOP blocks it)
- You can't access `iframe.contentDocument`
- But the user can *see* it rendered — this is how clickjacking works (the iframe is transparent, overlaid on fake UI)

---

## Exceptions to SOP

### `document.domain` (Legacy)
Two pages from subdomains of the same parent (e.g., `a.example.com` and `b.example.com`) can both set `document.domain = "example.com"` and then access each other's DOM.

**Security impact:** Deprecated and removed in modern browsers, but older apps may rely on it. A subdomain XSS could exploit this to attack the parent.

### Cross-Origin Messaging (`postMessage`)
Pages intentionally communicate across origins using `window.postMessage()`. The receiver should validate the `event.origin`.

**Security impact:** If the receiver doesn't check `event.origin`, an attacker can send malicious messages from any page.

### CORS
The server explicitly allows cross-origin reads via response headers. See [[CORS]].

---

## Why SOP Is Not Enough Alone

SOP stops reading, not sending. Three major attack classes exist because of this gap:

1. **CSRF** — attacker sends a state-changing request; doesn't need to read the response → [[CSRF]]
2. **Clickjacking** — attacker frames a legit page; user's clicks act on the real page → [[Browser-Security-Model]]
3. **XSS** — if attacker injects JS into the victim origin, SOP is irrelevant because the injected script IS same-origin → [[XSS-Overview]]

---

*Back to [[Browser-Security-Model]] | Related: [[CORS]] | [[CSRF]] | [[XSS-Overview]]*

*Sources: MDN Web Docs, PortSwigger Web Security Academy*
