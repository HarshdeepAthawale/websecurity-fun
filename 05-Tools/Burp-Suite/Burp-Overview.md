# Burp Suite — Overview and Setup

**Overview** | [[Burp-Intruder]] | [[Burp-Scanner]] | [[Burp-Extensions]]

---

Burp Suite is the standard tool for web application security testing. Everything you do in web security eventually goes through Burp. Learn it properly.

---

## What Burp Does

Burp sits between your browser and the internet as an intercepting proxy. Every HTTP request your browser makes passes through Burp — you can see it, modify it, replay it, or send it to other tools.

```
Browser → Burp Proxy → Internet
           ↓
      You see and modify everything here
```

---

## Setup (First Time)

### Install

Download from [portswigger.net/burp](https://portswigger.net/burp). Community edition is free and sufficient for learning.

### Configure Browser Proxy

Set your browser to proxy through `127.0.0.1:8080`.

**Firefox (recommended):**
1. Settings → Network Settings → Manual proxy configuration
2. HTTP Proxy: `127.0.0.1`, Port: `8080`
3. Check "Use this proxy for HTTPS"

Or use **FoxyProxy** extension — lets you switch proxy on/off easily.

### Install CA Certificate (for HTTPS)

```
1. With Burp running, browse to http://burpsuite (or http://burp)
2. Click "CA Certificate" → download
3. Firefox: Settings → Privacy → Certificates → Import
4. Chrome: Settings → Security → Manage Certificates → Import
```

Now Burp can intercept HTTPS traffic.

---

## Core Tools

### Proxy (Tab 1)

The core of Burp. Intercept mode on/off. All traffic appears in the HTTP history.

- **Intercept ON:** Each request pauses for you to review/modify before forwarding
- **Intercept OFF:** Requests pass through transparently but are logged in HTTP history

**HTTP History** is where you browse the app first — everything gets captured here for later analysis.

### Repeater

Right-click any request → "Send to Repeater". Replay the request as many times as you want with modifications.

This is where you do most of your manual testing:
- Change parameter values
- Test different inputs
- Modify headers
- Try payloads

**Shortcut:** `Ctrl+R` sends to Repeater. `Ctrl+Space` sends the request in Repeater.

### Intruder

Automated request modification and sending. Used for:
- Brute force (logins, OTPs, token IDs)
- Fuzzing parameters with wordlists
- Enumerating IDs (IDOR testing)

See [[Burp-Intruder]] for full usage.

### Scanner (Pro only)

Automated vulnerability scanning. Passive scanning happens automatically as you browse; active scanning sends probes.

Community edition doesn't have the active scanner. See [[Burp-Scanner]].

### Decoder

Encode/decode data in various formats: URL, HTML, Base64, hex, etc.

```
Paste encoded data → select encoding type → decoded
```

### Comparer

Compare two requests/responses side by side. Useful for spotting differences between "admin" and "user" responses, or "valid token" vs "invalid token".

### Logger

Logs all traffic even when the Proxy is not open. Good for passive recon when just browsing.

---

## Essential Workflows

### Workflow 1: Find and Test a Parameter

1. Browse the app with Proxy History recording
2. Find a request with an interesting parameter (search, ID, user input)
3. Right-click → Send to Repeater
4. Modify the parameter, send, observe response

### Workflow 2: Compare Two Users' Access

1. Log in as User A, capture a request to a sensitive endpoint
2. Right-click → Send to Repeater
3. Log in as User B in a different browser, copy their session cookie
4. Replace the cookie in Repeater, change the resource ID to User A's
5. Send — does User B get User A's data?

### Workflow 3: Test for Injection

1. Send a request with a parameter to Repeater
2. Try: `'`, `"`, `\`, `{{7*7}}`, `<script>alert(1)</script>`, `;sleep 5`
3. Look for: SQL errors, template output (49), XSS execution, response delay

### Workflow 4: Intercept and Modify

1. Turn Intercept ON
2. Perform the action in the browser (login, form submit)
3. The request pauses in Burp — modify what you need
4. Click "Forward" to send

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Send to Repeater |
| `Ctrl+I` | Send to Intruder |
| `Ctrl+U` | URL encode selection |
| `Ctrl+Shift+U` | URL decode selection |
| `Ctrl+B` | Base64 encode selection |
| `Ctrl+Shift+B` | Base64 decode selection |
| `Ctrl+Space` | Send request (in Repeater) |
| `Ctrl+Z` | Undo in request editor |

---

## Useful Settings

### Target Scope

Define what's in scope so Burp focuses on the target:
- Target → Scope → Add
- This controls what appears in Proxy History, what Scanner scans, etc.

### Match and Replace

Proxy → Options → Match and Replace:
- Auto-replace User-Agent
- Strip headers automatically
- Add headers to every request

### Session Handling Rules

Proxy → Options → Sessions:
- Automatically re-login if session expires during Intruder runs
- Macro to handle CSRF token refresh

---

*Related: [[Burp-Intruder]] | [[Burp-Extensions]] | [[Burp-Scanner]]*
