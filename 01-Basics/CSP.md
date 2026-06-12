# Content Security Policy (CSP)

← [[Browser-Security-Model]]

---

CSP is a response header that tells the browser what it's allowed to do on a page. It's the strongest client-side defense against XSS — but a weak CSP is often worse than no CSP because it creates a false sense of security.

---

## How CSP Works

The server sends a header:

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

The browser enforces it: any script not from `self` or `cdn.example.com` is **blocked and not executed**, even if injected via XSS.

---

## Key Directives

| Directive | Controls |
|-----------|---------|
| `default-src` | Fallback for any directive not explicitly set |
| `script-src` | Where JavaScript can be loaded from |
| `style-src` | Where CSS can be loaded from |
| `img-src` | Where images can be loaded from |
| `connect-src` | Where fetch/XHR/WebSocket can connect to |
| `frame-src` | What can be embedded in iframes |
| `frame-ancestors` | Who can embed this page (replaces `X-Frame-Options`) |
| `form-action` | Where forms can submit to |
| `base-uri` | Restricts `<base>` tag (prevents base tag injection) |
| `object-src` | Flash/plugins — should always be `'none'` |
| `report-uri` | Where CSP violations are reported (monitoring) |

---

## Source Values

| Value | Meaning |
|-------|---------|
| `'self'` | Same origin |
| `'none'` | Nothing allowed |
| `'unsafe-inline'` | Allow inline scripts/styles |
| `'unsafe-eval'` | Allow `eval()`, `setTimeout(string)`, etc. |
| `'nonce-<base64>'` | Allow specific inline scripts with matching nonce |
| `'sha256-<hash>'` | Allow specific inline scripts matching hash |
| `https://cdn.com` | Allow from this specific host |
| `https:` | Allow any HTTPS URL |
| `*` | Allow everything (wildcard) |

---

## Dangerous CSP Patterns

### `unsafe-inline`

```http
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

Allows all inline scripts — completely defeats the purpose of CSP for XSS. An injected `<script>alert(1)</script>` runs fine.

### `unsafe-eval`

```http
Content-Security-Policy: script-src 'self' 'unsafe-eval'
```

Allows `eval()`. Some XSS payloads require eval, so this weakens CSP significantly.

### Wildcard Host

```http
Content-Security-Policy: script-src *
```

Allows scripts from any origin — same as no CSP.

### JSONP Endpoint Bypass

If a trusted host in `script-src` has a JSONP endpoint (returns user-controlled data wrapped in a JS callback), an attacker can use it:

```
https://trusted-cdn.com/jsonp?callback=alert(1)//
```

This is a valid `<script src>` load from a trusted origin, but the content is attacker-controlled.

### `object-src` Not Set to `'none'`

Flash and plugins bypass script-src entirely. Always: `object-src 'none'`.

---

## Nonces — The Right Way

A **nonce** is a random per-request value. The server generates it, puts it in the CSP header, and adds it to every legitimate inline script:

```http
Content-Security-Policy: script-src 'nonce-r4nd0mV4lu3=='
```

```html
<script nonce="r4nd0mV4lu3==">
  // This runs
</script>

<script>
  // Injected by XSS — no nonce, blocked
</script>
```

For nonces to be secure:
- Must be **random and unpredictable** per request
- Must **not be leaked** in logs, cache, `Referer` header

---

## CSP Bypass Techniques

Even a well-intentioned CSP often has gaps:

| Bypass | Condition |
|--------|-----------|
| JSONP on whitelisted domain | Trusted domain has JSONP endpoint |
| Script gadgets (AngularJS) | `angular.js` whitelisted → `ng-app` + expressions = XSS |
| `base` tag injection | No `base-uri` directive → inject `<base href="https://attacker.com">` |
| `data:` URI | `script-src data:` allowed |
| `unsafe-inline` present | CSP is moot |
| Nonce leak in URL | If nonce in `Referer` header to a third-party resource |
| DOM clobbering | Bypass CSP logic by manipulating DOM before scripts run |

> [!TIP]
> When you find XSS on a site with CSP, don't give up. Check the policy at [csp-evaluator.withgoogle.com](https://csp-evaluator.withgoogle.com). It tells you what's bypassable.

---

## Evaluating a CSP

Paste the policy into [csp-evaluator.withgoogle.com](https://csp-evaluator.withgoogle.com) — it rates each directive and flags bypass risks automatically.

**Strong CSP example:**
```http
Content-Security-Policy: 
  default-src 'none';
  script-src 'nonce-{random}';
  style-src 'self';
  img-src 'self' data:;
  connect-src 'self';
  font-src 'self';
  object-src 'none';
  base-uri 'none';
  form-action 'self';
  frame-ancestors 'none'
```

---

*Back to [[Browser-Security-Model]] | Related: [[XSS-Overview]] | [[HTTP-Headers]]*

*Sources: MDN Web Docs, PortSwigger Web Security Academy, CSP Evaluator*
